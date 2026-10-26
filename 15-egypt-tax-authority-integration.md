# Phase 15 — Egypt Tax Authority Integration (e-Invoice + e-Receipt)

**Tags:** `[Backend]` `[API]` `[Security]`
**Depends on:** Phase 8 (invoices), Phase 11 (tenancy — each tenant has its own ETA credentials)
**You'll finish this phase with:** an architecture that submits real B2B e-Invoices and B2C
e-Receipts to Egypt's Tax Authority (ETA), built behind an interface so KSA/UAE/other MENA tax
authorities can be added later without touching invoicing or ledger code.

---

## ⚠️ Read this before writing any code

This is the one phase in the whole roadmap where **the code is the easy part.** Before any of
this works against the real ETA system, your business needs, outside of code:
1. **Registration on the ETA e-invoicing portal** as a taxpayer, which issues API credentials.
2. **A digital signing certificate on a hardware USB token** (a "smart card" token — commonly
   sold by Egyptian Trust/CA providers) — this is a physical device, not something that can be
   generated in software.
3. Access to ETA's **pre-production (sandbox) environment** first — get exact sandbox URLs and
   the current technical specification from ETA's own developer portal, since these details (and
   validation rules) are updated by the tax authority itself and can change independently of
   anything in this codebase.

I'm building the architecture and the integration skeleton in this phase. Treat every field name
and endpoint below as "this is the shape of it" — confirm the exact current specification against
ETA's official documentation before going live, the same way you'd never hardcode a bank's SWIFT
format from a blog post without checking the bank's own spec.

---

## 1. Learning objectives

- Build an integration behind an **interface**, so a second tax authority (Phase-roadmap-adjacent:
  Saudi ZATCA) is a new class, not a rewrite.
- Understand a real constraint most tutorials skip: **hardware-token signing doesn't fit neatly
  into a cloud backend**, and design around it honestly instead of pretending it's a simple API
  call.
- Queue the submission instead of making the user's "save invoice" click wait on a government
  server's response time.

## 2. The real-world flow, and the architectural wrinkle

ETA's e-invoicing flow: your app builds the invoice as a JSON document (issuer/receiver details,
an ETA activity code, each line item's classification via a GS1 or Egypt-specific EGS code, unit
codes from UN/ECE Recommendation 20, tax breakdown, and the EGP equivalent for any foreign-currency
amount) → the document is **digitally signed** (CAdES-BES detached signature) using the
taxpayer's certificate → the signed payload is submitted to ETA's API → ETA responds with either
acceptance (a UUID and a "long ID") or rejection with error codes. **e-Receipts for B2C sales use
a separate, simpler schema** — don't submit consumer sales through the B2B invoice schema; they
have different validation rules and reporting categories.

**The wrinkle:** the signing certificate lives on a physical USB token. A cloud-hosted Laravel
backend, running on a server in a datacenter, has no USB port for your client's token to plug
into. Two realistic options:
- **A local signing agent**: a small piece of software running on the accountant's own PC (the
  one the USB token is plugged into) that receives an unsigned document from your cloud API,
  signs it locally using the token, and posts the signed result back. This is the most common
  real-world pattern for USB-token-based e-invoicing integrations.
- **A cloud/remote signing service**, if your certificate provider offers one (some Egyptian CAs
  do) — this removes the local-agent requirement but depends on that provider's product.

Design the `TaxAuthority` interface so either option can implement `sign()` — don't assume signing
happens inline in the same process as the rest of your Laravel app.

---

## 3. Schema

```php
Schema::create('tax_submissions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('invoice_id')->nullable()->constrained();
    $table->enum('document_type', ['invoice', 'receipt']); // B2B vs B2C — different ETA schemas
    $table->enum('status', ['pending', 'submitted', 'accepted', 'rejected'])->default('pending');
    $table->string('uuid')->nullable();      // assigned by ETA on acceptance
    $table->string('long_id')->nullable();   // assigned by ETA on acceptance
    $table->json('request_payload')->nullable();
    $table->json('response_payload')->nullable();
    $table->text('error_message')->nullable();
    $table->unsignedTinyInteger('attempts')->default(0);
    $table->timestamps();
});
```

---

## 4. The interface, so this isn't Egypt-only forever

```php
interface TaxAuthorityInterface
{
    public function submitInvoice(Invoice $invoice): TaxSubmissionResult;
    public function submitReceipt(Invoice $invoice): TaxSubmissionResult;
    public function checkStatus(string $uuid): TaxSubmissionResult;
}

// A plain DTO, not an Eloquent model — this is data passed between the
// tax-authority adapter and the code that calls it, deliberately decoupled
// from how `tax_submissions` happens to be stored.
class TaxSubmissionResult
{
    public function __construct(
        public readonly string $status,       // 'accepted' | 'rejected' | 'pending'
        public readonly ?string $uuid = null,
        public readonly ?string $longId = null,
        public readonly ?string $errorMessage = null,
        public readonly array $rawResponse = [],
    ) {}
}
```

`app/Services/TaxAuthority/EgyptEtaService.php`:
```php
class EgyptEtaService implements TaxAuthorityInterface
{
    public function __construct(
        private EtaDocumentBuilder $builder,   // turns an Invoice into ETA's JSON shape
        private EtaSigningClient $signer,      // talks to the local agent or remote signer
        private Client $http,                  // Guzzle, configured with this tenant's ETA base URL + credentials
    ) {}

    public function submitInvoice(Invoice $invoice): TaxSubmissionResult
    {
        $document = $this->builder->buildInvoiceDocument($invoice);
        $signed = $this->signer->sign($document);

        try {
            $response = $this->http->post('/api/v1/documentsubmissions', ['json' => $signed]);
            $body = json_decode($response->getBody(), true);

            return new TaxSubmissionResult(
                status: 'accepted',
                uuid: $body['acceptedDocuments'][0]['uuid'] ?? null,
                longId: $body['acceptedDocuments'][0]['longId'] ?? null,
                rawResponse: $body,
            );
        } catch (RequestException $e) {
            $body = json_decode($e->getResponse()?->getBody() ?? '{}', true);
            return new TaxSubmissionResult(
                status: 'rejected',
                errorMessage: $body['error']['details'][0]['message'] ?? $e->getMessage(),
                rawResponse: $body,
            );
        }
    }

    public function submitReceipt(Invoice $invoice): TaxSubmissionResult
    {
        // Same shape, but $this->builder->buildReceiptDocument() and a different
        // endpoint — the two schemas are genuinely different documents to ETA,
        // not just a flag on one document type. Never conflate them.
    }

    public function checkStatus(string $uuid): TaxSubmissionResult { /* GET the document by uuid */ }
}
```

Bind it in a service provider, keyed so it's swappable per tenant/country later:
```php
$this->app->bind(TaxAuthorityInterface::class, function () {
    return match (currentTenant()->country_code) {
        'EG' => app(EgyptEtaService::class),
        // 'SA' => app(SaudiZatcaService::class),  // added later, same interface
        default => throw new \RuntimeException('No tax authority integration configured for this country.'),
    };
});
```

---

## 5. Queue it — never block the user on a government API's response time

```php
class SubmitInvoiceToTaxAuthority implements ShouldQueue
{
    public $tries = 5;
    public $backoff = [60, 300, 900, 3600, 14400]; // widening retry gaps

    public function __construct(private int $invoiceId, private int $tenantId) {}

    public function handle(TaxAuthorityInterface $taxAuthority): void
    {
        // Required because this runs outside an HTTP request (Phase 11's tenancy warning) —
        // without this line, TenantScope would see no tenant and could leak data.
        app()->instance('currentTenantId', $this->tenantId);

        $invoice = Invoice::findOrFail($this->invoiceId);
        $result = $taxAuthority->submitInvoice($invoice);

        TaxSubmission::updateOrCreate(
            ['invoice_id' => $invoice->id, 'document_type' => 'invoice'],
            [
                'status' => $result->status,
                'uuid' => $result->uuid,
                'long_id' => $result->longId,
                'error_message' => $result->errorMessage,
                'response_payload' => $result->rawResponse,
            ]
        );

        if ($result->status === 'rejected') {
            // A validation rejection (bad activity code, wrong tax number) won't fix
            // itself on retry — fail loudly instead of silently retrying 5 times.
            $this->fail(new \RuntimeException($result->errorMessage));
        }
    }
}
```
Dispatch it from `InvoiceService::create()` (Phase 8), right after `$this->postingService->post($entry)`:
```php
SubmitInvoiceToTaxAuthority::dispatch($invoice->id, currentTenantId());
```

---

## 6. What you built, and why it matters later

- A **swappable adapter**, not an Egypt-shaped hardcoded integration — the interface is what lets
  this project genuinely serve "MENA, starting with Egypt" as the roadmap intended.
- An honest architecture for the USB-token signing constraint, instead of code that would silently
  fail the first time it meets a real hardware token.
- The retry/backoff/queue pattern here is the exact template Phase 18 generalizes for recurring
  invoices and other background work.

## 7. Next: Phase 16

With tax compliance handled, Phase 16 builds cheque printing — precise PDF templates matched to
real Egyptian bank cheque leaf layouts.
