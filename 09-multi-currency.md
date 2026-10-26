# Phase 9 — Multi-Currency

**Tags:** `[Backend]`
**Depends on:** Phase 8 (invoices/payments to attach currency to)
**You'll finish this phase with:** invoices and journal entries that can be issued in any
supported currency, always store their equivalent in your base currency (EGP), and a scheduled
job keeping exchange rates current.

---

## 1. Learning objectives

- Understand why accounting systems store **both** the original transaction amount *and* its
  converted base-currency amount, not just one.
- Practice a scheduled background job (a preview of the full cron/queue system in Phase 18).

## 2. Why this phase now

Invoicing (Phase 8) exists; MENA-wide (not just Egypt) means customers who bill in USD or EUR
need to appear correctly on EGP-denominated financial reports (Phase 10). This has to be right
before reports are built on top of it, or every report gets rebuilt afterward.

---

## 3. The principle: dual-amount storage

Never convert-and-discard. Every money column that can be foreign-currency stores **both**:
- the amount in the transaction's own currency, and
- the amount in your base/functional currency (EGP for a business operating in Egypt), using the
  exchange rate **at the time of the transaction** — not today's rate.

This matters because exchange rates move daily; if you only stored the original amount and
converted on every report, an invoice from January would report a different EGP value in March
than it did in January, which is not how real bookkeeping (or auditors) work.

---

## 4. Schema

```php
// currencies
Schema::create('currencies', function (Blueprint $table) {
    $table->id();
    $table->string('code', 3)->unique(); // ISO 4217: EGP, USD, EUR, SAR...
    $table->string('name');
    $table->string('symbol');
    $table->boolean('is_base')->default(false); // exactly one row is true (EGP)
    $table->timestamps();
});

// exchange_rates — one row per currency per day
Schema::create('exchange_rates', function (Blueprint $table) {
    $table->id();
    $table->foreignId('currency_id')->constrained();
    $table->date('rate_date');
    $table->decimal('rate_to_base', 15, 6); // e.g. 1 USD = 48.500000 EGP
    $table->timestamps();
    $table->unique(['currency_id', 'rate_date']);
});
```

Add to `invoices` (migration `add_currency_to_invoices_table`):
```php
$table->foreignId('currency_id')->constrained();
$table->decimal('exchange_rate', 15, 6)->default(1); // rate at issue_date, frozen forever after
$table->decimal('total_base_currency', 15, 2); // total * exchange_rate, stored, not recomputed
```
Same two columns (`exchange_rate`, and a `_base_currency` equivalent of every money column) go on
`journal_entry_lines` too, since that's what the ledger and reports actually read from.

---

## 5. `FxRateService` and the scheduled job

```php
class FxRateService
{
    public function refreshRates(): void
    {
        $base = Currency::where('is_base', true)->firstOrFail(); // EGP

        foreach (Currency::where('is_base', false)->get() as $currency) {
            // Any FX API works here — this is intentionally provider-agnostic.
            // Wrap the actual HTTP call so swapping providers later is a one-file change.
            $rate = app(FxProviderInterface::class)->getRate($currency->code, $base->code);

            ExchangeRate::updateOrCreate(
                ['currency_id' => $currency->id, 'rate_date' => now()->toDateString()],
                ['rate_to_base' => $rate]
            );
        }
    }
}
```

```powershell
php artisan make:command RefreshExchangeRates
```
```php
class RefreshExchangeRates extends Command
{
    protected $signature = 'app:refresh-exchange-rates';

    public function handle(FxRateService $service): void
    {
        $service->refreshRates();
        $this->info('Exchange rates updated for ' . now()->toDateString());
    }
}
```

Schedule it, `routes/console.php`:
```php
Schedule::command('app:refresh-exchange-rates')->dailyAt('06:00');
```
*(Phase 18 covers actually running the scheduler as a background service — for now, you can run
`php artisan app:refresh-exchange-rates` manually to test it.)*

---

## 6. Using it in `InvoiceService`

Update Phase 8's `InvoiceService::create()` to look up and freeze the rate at creation time:
```php
$currency = Currency::findOrFail($data['currency_id']);
$rate = $currency->is_base ? 1 : ExchangeRate::where('currency_id', $currency->id)
    ->where('rate_date', '<=', $data['issue_date'])
    ->latest('rate_date')->value('rate_to_base') ?? 1;

$invoice = Invoice::create([
    // ...existing fields from Phase 8
    'currency_id' => $currency->id,
    'exchange_rate' => $rate,
    'total_base_currency' => $total * $rate,
]);

// journal_entry_lines then store debit/credit in base currency (what the ledger has always
// used), while invoice_lines keep the original transaction-currency amounts for the printed
// invoice/PDF the customer sees.
```

*Why freeze the rate on the invoice, not on each payment against it:* the revenue was earned, and
the receivable created, at the exchange rate on the **issue date**. Any FX gain/loss between issue
and payment is a real, separate accounting event (a topic for a later, dedicated "FX
gain/loss" journal entry) — not something to silently absorb by re-converting the original amount.

---

## 7. What you built, and why it matters later

- **Dual-amount storage** as a standing rule for every money column from here on — cheque
  printing (Phase 16) prints the transaction-currency amount; reports (Phase 10) always sum base
  currency.
- A first, small, real **scheduled job** — the exact pattern (a `FxProviderInterface`, a
  Command, a `Schedule::command()` line) that Phase 18 generalizes for recurring invoices and tax
  submission retries.

## 8. Next: Phase 10

With currency handled correctly, Phase 10 builds the reports that read the ledger: Trial Balance,
Income Statement, Balance Sheet, and Cash Flow.
