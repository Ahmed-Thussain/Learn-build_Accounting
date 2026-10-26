# Phase 18 — Cron Jobs & Queues

**Tags:** `[Backend]`
**Depends on:** Phases 9, 15, 17 (all referenced background jobs loosely — this phase makes the
infrastructure real)
**You'll finish this phase with:** a properly running queue worker and scheduler, on Windows,
plus a recurring-invoice job as one more concrete example of the pattern.

---

## 1. Learning objectives

- Understand the difference between the **Scheduler** (Laravel decides *when* something runs —
  "every day at 6am") and the **Queue** (Laravel decides *how* a unit of work runs — in the
  background, with retries, off the request/response cycle).
- Get both running reliably in **local Windows development**, where there's no native cron.

## 2. Why this phase now

Phases 9, 15, and 17 all said "this becomes a queued job" without showing the actual worker
process. Time to make that real, and to establish the one pattern every background job in this app
should follow — including tenancy safety, which every earlier job-related phase already flagged.

---

## 3. Queues: configure and run

`.env`:
```env
QUEUE_CONNECTION=database
```
```powershell
php artisan queue:table
php artisan migrate
```

Run a worker (a long-lived process that pulls jobs and executes them):
```powershell
php artisan queue:work --tries=3
```

*Why `QUEUE_CONNECTION=database` to start, not Redis:* it needs zero extra infrastructure — the
jobs table already lives in the MySQL database Laragon gave you in Phase 1. Move to Redis
(`composer require predis/predis`, `QUEUE_CONNECTION=redis`) once job volume is high enough that
polling the database becomes a real bottleneck — not before, since that's added operational
complexity you don't need yet.

---

## 4. Scheduling: the recurring-invoice example

A common real feature: some customers get invoiced automatically every month.

```php
Schema::create('recurring_invoice_templates', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained();
    $table->foreignId('contact_id')->constrained('contacts');
    $table->json('line_items'); // same shape as invoice_lines input
    $table->foreignId('currency_id')->constrained();
    $table->enum('frequency', ['weekly', 'monthly']);
    $table->date('next_run_date');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});
```

```powershell
php artisan make:command GenerateRecurringInvoices
```
```php
class GenerateRecurringInvoices extends Command
{
    protected $signature = 'app:generate-recurring-invoices';

    public function handle(): void
    {
        // This command runs with NO tenant context (it's not tied to any one
        // request), so it must loop tenant-by-tenant and set context explicitly
        // for each — exactly the tenancy rule flagged back in Phase 11.
        Tenant::each(function ($tenant) {
            app()->instance('currentTenantId', $tenant->id);

            RecurringInvoiceTemplate::where('is_active', true)
                ->where('next_run_date', '<=', now()->toDateString())
                ->each(function ($template) {
                    GenerateInvoiceFromTemplate::dispatch($template->id, $template->tenant_id);

                    $template->update([
                        'next_run_date' => match ($template->frequency) {
                            'weekly' => now()->addWeek()->toDateString(),
                            'monthly' => now()->addMonthNoOverflow()->toDateString(),
                        },
                    ]);
                });
        });
    }
}
```
```php
class GenerateInvoiceFromTemplate implements ShouldQueue
{
    public function __construct(private int $templateId, private int $tenantId) {}

    public function handle(InvoiceService $invoiceService): void
    {
        app()->instance('currentTenantId', $this->tenantId);
        $template = RecurringInvoiceTemplate::findOrFail($this->templateId);

        $invoiceService->create([
            'contact_id' => $template->contact_id,
            'issue_date' => now()->toDateString(),
            'due_date' => now()->addDays(30)->toDateString(),
            'currency_id' => $template->currency_id,
            'lines' => $template->line_items,
        ]);
    }
}
```

`routes/console.php` — every scheduled task in the app, in one file:
```php
Schedule::command('app:refresh-exchange-rates')->dailyAt('06:00');       // Phase 9
Schedule::command('app:generate-recurring-invoices')->dailyAt('01:00');
```

---

## 5. Running this reliably on Windows during development

Windows has no native cron. Two workable options while developing:

**Option A — a long-running scheduler process** (simplest for local dev):
```powershell
php artisan schedule:work
```
This blocks the terminal and checks every minute whether any scheduled command is due — leave it
running alongside `php artisan serve`, `npm run dev`, and `php artisan queue:work` (four terminals
total during development; Phase 21 covers running all of these properly as background services in
production via a process manager).

**Option B — Windows Task Scheduler**, calling `php artisan schedule:run` every minute — closer to
how a real Linux cron-based production server would be configured, useful if you want your local
setup to mirror production behavior exactly.

---

## 6. What you built, and why it matters later

- A real, running background-processing setup — not just referenced code from earlier phases.
- The **"loop tenants explicitly, set context per tenant"** pattern demonstrated concretely for
  the first time — every scheduled command from here on (including anything Phase 19's autonomous
  agents run on a timer) copies this shape exactly.

## 7. Next: Phase 19

With reliable background processing in place, Phase 19 builds the AI assistant and autonomous
agents — including the permission boundaries and audit trail that make giving an AI the ability to
act inside real financial data safe.
