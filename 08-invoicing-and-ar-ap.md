# Phase 8 — Invoicing & Accounts Receivable/Payable

**Tags:** `[Backend]` `[Frontend]` `[API]`
**Depends on:** Phase 4 (posting engine), Phase 7 (UI patterns)
**You'll finish this phase with:** creating an invoice automatically posts the correct journal
entry behind the scenes, and recording a payment against it does the same — a non-accountant user
never touches a debit/credit field directly.

---

## 1. Learning objectives

- Practice the core pattern of "business document" modules in this app: **a domain action
  (create invoice) generates a journal entry, it never bypasses it.**
- Understand accrual accounting basics: an invoice is revenue **when issued**, not when paid;
  a payment reduces what's owed, it's a separate event.

## 2. Why this phase now

The ledger and posting engine (Phase 4) are solid. This phase is the first real "business layer"
on top of them, and every later domain feature (cheque payments, e-commerce orders) is really just
another way of creating an Invoice or a Payment — so this pattern needs to be right.

---

## 3. Schema

```powershell
php artisan make:migration create_invoices_table
php artisan make:migration create_invoice_lines_table
php artisan make:migration create_payments_table
```

```php
// invoices
Schema::create('invoices', function (Blueprint $table) {
    $table->id();
    $table->string('invoice_number')->unique();
    $table->foreignId('contact_id')->constrained('contacts');
    $table->date('issue_date');
    $table->date('due_date');
    $table->enum('status', ['draft', 'sent', 'partially_paid', 'paid', 'void'])->default('draft');
    $table->decimal('total', 15, 2);
    $table->foreignId('journal_entry_id')->nullable()->constrained('journal_entries');
    $table->timestamps();
    $table->softDeletes();
});

// invoice_lines
Schema::create('invoice_lines', function (Blueprint $table) {
    $table->id();
    $table->foreignId('invoice_id')->constrained()->cascadeOnDelete();
    $table->string('description');
    $table->decimal('quantity', 10, 2)->default(1);
    $table->decimal('unit_price', 15, 2);
    $table->decimal('line_total', 15, 2); // quantity * unit_price, stored (not recomputed on read)
    $table->timestamps();
});

// payments
Schema::create('payments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('invoice_id')->constrained();
    $table->date('paid_on');
    $table->decimal('amount', 15, 2);
    $table->enum('method', ['cash', 'bank_transfer', 'cheque'])->default('cash');
    $table->foreignId('journal_entry_id')->nullable()->constrained('journal_entries');
    $table->timestamps();
});
```

*Why `journal_entry_id` on both `invoices` and `payments`:* every financial document in this app
traces back to the ledger entry it created. This is what makes "show me the journal entry behind
this invoice" a one-line query instead of forensic work, and it's the same traceability the ETA
tax integration (Phase 15) and the AI audit trail (Phase 19) both lean on.

---

## 4. `InvoiceService` — the pattern in action

```php
class InvoiceService
{
    public function __construct(private PostingService $postingService) {}

    public function create(array $data): Invoice
    {
        return DB::transaction(function () use ($data) {
            $total = collect($data['lines'])->sum(fn ($l) => $l['quantity'] * $l['unit_price']);

            $invoice = Invoice::create([
                'invoice_number' => 'INV-' . now()->format('Y') . '-' . str_pad(Invoice::count() + 1, 6, '0', STR_PAD_LEFT),
                'contact_id' => $data['contact_id'],
                'issue_date' => $data['issue_date'],
                'due_date' => $data['due_date'],
                'status' => 'sent',
                'total' => $total,
            ]);

            foreach ($data['lines'] as $line) {
                $invoice->lines()->create([
                    ...$line,
                    'line_total' => $line['quantity'] * $line['unit_price'],
                ]);
            }

            // The accounting rule this whole method exists to enforce:
            // issuing an invoice = Debit Accounts Receivable, Credit Sales Revenue.
            // A user filling in "customer, items, price" never sees this — but it
            // is exactly as real, and exactly as balanced, as a manual journal entry.
            $entry = JournalEntry::create([
                'reference_number' => 'JE-' . $invoice->invoice_number,
                'entry_date' => $invoice->issue_date,
                'description' => "Invoice {$invoice->invoice_number}",
                'status' => 'draft',
                'created_by' => auth()->id(),
            ]);
            $entry->lines()->create([
                'account_id' => Account::where('code', '1100')->value('id'), // Accounts Receivable
                'contact_id' => $invoice->contact_id,
                'debit' => $total, 'credit' => 0,
            ]);
            $entry->lines()->create([
                'account_id' => Account::where('code', '4000')->value('id'), // Sales Revenue
                'debit' => 0, 'credit' => $total,
            ]);

            $this->postingService->post($entry);
            $invoice->update(['journal_entry_id' => $entry->id]);

            return $invoice->load('lines');
        });
    }
}
```

`PaymentService`, the mirror image (Debit Cash, Credit Accounts Receivable):

```php
class PaymentService
{
    public function __construct(private PostingService $postingService) {}

    public function record(Invoice $invoice, array $data): Payment
    {
        return DB::transaction(function () use ($invoice, $data) {
            $payment = $invoice->payments()->create([
                'paid_on' => $data['paid_on'],
                'amount' => $data['amount'],
                'method' => $data['method'],
            ]);

            $entry = JournalEntry::create([
                'reference_number' => 'JE-PMT-' . $payment->id,
                'entry_date' => $payment->paid_on,
                'description' => "Payment for {$invoice->invoice_number}",
                'status' => 'draft',
                'created_by' => auth()->id(),
            ]);
            $entry->lines()->create([
                'account_id' => Account::where('code', '1010')->value('id'), // Bank
                'debit' => $payment->amount, 'credit' => 0,
            ]);
            $entry->lines()->create([
                'account_id' => Account::where('code', '1100')->value('id'), // Accounts Receivable
                'contact_id' => $invoice->contact_id,
                'debit' => 0, 'credit' => $payment->amount,
            ]);
            $this->postingService->post($entry);
            $payment->update(['journal_entry_id' => $entry->id]);

            $totalPaid = $invoice->payments()->sum('amount');
            $invoice->update([
                'status' => bccomp((string) $totalPaid, (string) $invoice->total, 2) >= 0
                    ? 'paid' : 'partially_paid',
            ]);

            return $payment;
        });
    }
}
```

---

## 5. API + Controller

Following Phase 5's exact template (`StoreInvoiceRequest`, `InvoicePolicy`, `InvoiceResource`):

```php
Route::apiResource('invoices', InvoiceController::class);
Route::post('/invoices/{invoice}/payments', [PaymentController::class, 'store']);
```

```php
class InvoiceController extends Controller
{
    public function __construct(private InvoiceService $invoiceService) {}

    public function store(StoreInvoiceRequest $request)
    {
        $invoice = $this->invoiceService->create($request->validated());
        return new InvoiceResource($invoice);
    }
    // index/show follow Phase 5's pattern exactly
}
```

---

## 6. Frontend

`frontend/src/views/InvoicesView.vue` — a list + "New Invoice" form reusing the exact
line-items-array pattern from Phase 7's journal entry form (contact picker + product/description
rows instead of account rows), and a "Record Payment" modal per unpaid invoice. Structurally
identical to Phase 7's component — deliberately not re-typed here in full; copy
`JournalEntryForm.vue`'s shape and swap the fields.

---

## 7. What you built, and why it matters later

- Confirmed the pattern: **every business action creates a journal entry through `PostingService`,
  never around it.** The e-commerce connector (Phase 17) will call `InvoiceService::create()`
  directly from a webhook — it doesn't need its own accounting logic, because this service already
  guarantees correctness.
- `journal_entry_id` traceability from a business document back to the ledger — required for
  Phase 15's tax authority submissions, which report on invoices, not raw ledger lines.

## 8. Next: Phase 9

Right now every amount is implicitly in one currency. Phase 9 adds real multi-currency support —
storing both the transaction currency and its converted value, and a daily job to keep exchange
rates current.
