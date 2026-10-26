# Phase 17 — E-Commerce Extension

**Tags:** `[Backend]` `[API]`
**Depends on:** Phase 8 (InvoiceService), Phase 11 (tenancy)
**You'll finish this phase with:** a webhook receiver that turns a real WooCommerce order into an
invoice in your accounting system automatically, using the exact same `InvoiceService` a human
user's form submission uses.

---

## 1. Learning objectives

- Understand a **connector** as a translation layer, not a new accounting subsystem — its whole
  job is "map their data shape onto our domain objects," then call the services that already exist.
- Verify **webhook authenticity** (HMAC signatures) — a webhook endpoint is a public URL; anyone
  can POST to it pretending to be your store unless you check.

## 2. Why this phase now

`InvoiceService::create()` (Phase 8) doesn't know or care whether it was called from a Vue form, a
Tinker session, or a webhook — that's by design. This phase is proof of that design: the entire
"e-commerce integration" is a translation function plus a webhook route, nothing more.

---

## 3. Receiving the webhook, safely

```php
Schema::create('ecommerce_connections', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained();
    $table->enum('platform', ['woocommerce', 'shopify']);
    $table->string('store_url');
    $table->string('webhook_secret'); // used to verify the HMAC signature below
    $table->timestamps();
});
```

```php
class WooCommerceWebhookController extends Controller
{
    public function handleOrderCreated(Request $request)
    {
        $connection = EcommerceConnection::where('store_url', $request->header('X-WC-Webhook-Source'))
            ->firstOrFail();

        // Verify the payload actually came from this store, not from anyone who
        // discovered the URL. WooCommerce signs the raw body with HMAC-SHA256
        // using the secret you configured when setting up the webhook.
        $signature = base64_encode(hash_hmac('sha256', $request->getContent(), $connection->webhook_secret, true));
        if (! hash_equals($signature, $request->header('X-WC-Webhook-Signature', ''))) {
            abort(401, 'Invalid webhook signature.');
        }

        // The webhook context has no logged-in user — set tenancy explicitly,
        // exactly like Phase 15's queued job had to.
        app()->instance('currentTenantId', $connection->tenant_id);

        ProcessEcommerceOrder::dispatch($connection->id, $request->all());

        return response()->json(['status' => 'queued']);
    }
}
```
*Why queue it rather than process inline:* respond to the webhook fast (WooCommerce/Shopify will
retry if you're slow or return an error) — the actual invoice creation, which touches the ledger,
happens safely in the background where a retry-on-failure job (Phase 18's pattern again) can
handle transient issues.

---

## 4. The translation layer — the actual "connector"

```php
class WooCommerceOrderMapper
{
    public function __construct(private InvoiceService $invoiceService) {}

    public function toInvoice(array $wooOrderPayload): Invoice
    {
        $contact = Contact::firstOrCreate(
            ['email' => $wooOrderPayload['billing']['email']],
            [
                'type' => 'customer',
                'name' => $wooOrderPayload['billing']['first_name'] . ' ' . $wooOrderPayload['billing']['last_name'],
                'phone' => $wooOrderPayload['billing']['phone'] ?? null,
                'address' => $wooOrderPayload['billing']['address_1'] ?? null,
            ]
        );

        $lines = collect($wooOrderPayload['line_items'])->map(fn ($item) => [
            'description' => $item['name'],
            'quantity' => $item['quantity'],
            'unit_price' => $item['price'],
        ])->toArray();

        // The whole point of this phase: an order from a completely different
        // system becomes, at this line, exactly the same call Phase 7's UI form
        // makes. Every accounting rule (Phase 4), every currency rule (Phase 9),
        // every tax submission (Phase 15) triggered by InvoiceService::create()
        // applies identically here — nothing about "it came from WooCommerce"
        // needs special-casing anywhere else in the app.
        return $this->invoiceService->create([
            'contact_id' => $contact->id,
            'issue_date' => now()->toDateString(),
            'due_date' => now()->toDateString(),
            'currency_id' => Currency::where('code', $wooOrderPayload['currency'])->value('id'),
            'lines' => $lines,
        ]);
    }
}
```

```php
class ProcessEcommerceOrder implements ShouldQueue
{
    public function __construct(private int $connectionId, private array $payload) {}

    public function handle(WooCommerceOrderMapper $mapper): void
    {
        $connection = EcommerceConnection::findOrFail($this->connectionId);
        app()->instance('currentTenantId', $connection->tenant_id);
        $mapper->toInvoice($this->payload);
    }
}
```

---

## 5. What you built, and why it matters later

- Confirmed, for the second time (after Phase 15's tax-authority queue), that **the domain
  services built in Phase 4/8 are the real product** — every "integration" phase in this roadmap
  is thin because the core was built right.
- A pattern (verify signature → queue → translate → call existing service) that a future Shopify
  connector, or any other external data source, copies directly.

## 6. Next: Phase 18

Several phases now have background jobs (FX rates, tax submission, e-commerce orders) referenced
loosely. Phase 18 builds the actual scheduler/queue infrastructure properly, including how to run
it reliably on Windows during development.
