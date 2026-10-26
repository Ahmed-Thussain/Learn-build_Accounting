# Phase 4 — The Double-Entry Accounting Engine

**Tags:** `[Backend]`
**Depends on:** Phase 2 (schema), Phase 3 (auth)
**You'll finish this phase with:** a `PostingService` that is the *only* legal way to post a
journal entry, guaranteeing every posted entry balances and can never silently be edited afterward.

---

## 1. Learning objectives

- Understand why business rules like "debits must equal credits" belong in a **service class**,
  not scattered across controllers.
- Understand **immutability** as an accounting principle: once posted, a record is corrected by
  *reversing* it, never by editing it.
- Practice wrapping multi-step writes in a **database transaction**.

## 2. Why this phase now

This is the single riskiest piece of logic in the whole app — get it wrong and every report,
every tax submission, and every AI action built on top of it later inherits the bug. It goes
right after auth and before *any* UI or API layer, so it can be reasoned about and tested in
isolation first.

---

## 3. The rule, precisely

A journal entry may be posted **only if**:

1. It has at least two lines.
2. `SUM(debit) === SUM(credit)` across all its lines, to the cent.
3. It is currently in `draft` status (you can't post an already-posted or void entry).

Once posted, no line, amount, or account on that entry may ever change. Correcting a mistake
means creating a **new** journal entry with debits/credits reversed, linked via
`reverses_journal_entry_id` (added in Phase 2's migration) — the original stays in the books,
visibly voided, forever. This is not paranoia — it's what makes an accounting system's history
trustworthy to an auditor, and it's the same guarantee we'll lean on when the AI agent (Phase 19)
is allowed to post entries: it can create new corrections, it can never rewrite history.

---

## 4. The exception and the service

```powershell
php artisan make:exception UnbalancedJournalEntryException
```

```php
class UnbalancedJournalEntryException extends Exception
{
    public function __construct(float $totalDebit, float $totalCredit)
    {
        parent::__construct(
            "Journal entry does not balance: debits {$totalDebit} != credits {$totalCredit}"
        );
    }
}
```

`app/Services/PostingService.php`:

```php
class PostingService
{
    /**
     * The only method in the whole codebase allowed to move a journal entry
     * from draft to posted. Controllers, jobs, the AI agent, the e-commerce
     * connector — everything routes through here.
     */
    public function post(JournalEntry $entry): JournalEntry
    {
        if ($entry->status !== 'draft') {
            throw new \RuntimeException("Only draft entries can be posted (this entry is {$entry->status}).");
        }

        $lines = $entry->lines;

        if ($lines->count() < 2) {
            throw new \RuntimeException('A journal entry needs at least two lines.');
        }

        $totalDebit = $lines->sum('debit');
        $totalCredit = $lines->sum('credit');

        // bccomp compares the two numbers as strings, avoiding the floating-point
        // rounding errors a plain `!==` comparison on decimals can produce.
        if (bccomp((string) $totalDebit, (string) $totalCredit, 2) !== 0) {
            throw new UnbalancedJournalEntryException($totalDebit, $totalCredit);
        }

        // The whole operation succeeds or fails as one unit — if anything after
        // this point throws, the status change is rolled back with it.
        return DB::transaction(function () use ($entry) {
            $entry->update(['status' => 'posted']);
            return $entry->fresh('lines');
        });
    }

    /**
     * Corrects a posted entry by creating a new entry with every line's
     * debit/credit swapped — never edits the original.
     */
    public function reverse(JournalEntry $entry, string $reason): JournalEntry
    {
        if ($entry->status !== 'posted') {
            throw new \RuntimeException('Only posted entries can be reversed.');
        }

        return DB::transaction(function () use ($entry, $reason) {
            $reversal = JournalEntry::create([
                'reference_number' => $entry->reference_number . '-REV',
                'entry_date' => now()->toDateString(),
                'description' => "Reversal of {$entry->reference_number}: {$reason}",
                'status' => 'draft',
                'created_by' => auth()->id(),
                'reverses_journal_entry_id' => $entry->id,
            ]);

            foreach ($entry->lines as $line) {
                $reversal->lines()->create([
                    'account_id' => $line->account_id,
                    'contact_id' => $line->contact_id,
                    'debit' => $line->credit,   // swapped
                    'credit' => $line->debit,   // swapped
                    'description' => "Reversal: {$line->description}",
                ]);
            }

            $this->post($reversal);
            $entry->update(['status' => 'void']);

            return $reversal;
        });
    }
}
```

---

## 5. The ledger read-side

`app/Services/LedgerService.php` — a first, small version; Phase 10 builds full reports on top of
this same idea:

```php
class LedgerService
{
    public function balanceFor(Account $account, ?string $asOfDate = null): float
    {
        $query = $account->lines()
            ->whereHas('journalEntry', fn ($q) => $q->where('status', 'posted'));

        if ($asOfDate) {
            $query->whereHas('journalEntry', fn ($q) => $q->where('entry_date', '<=', $asOfDate));
        }

        $debit = (clone $query)->sum('debit');
        $credit = (clone $query)->sum('credit');

        // Asset/expense accounts increase with debits; liability/equity/revenue
        // accounts increase with credits. This sign flip is *the* rule that makes
        // a trial balance and balance sheet read correctly — get it backwards and
        // every report silently shows the opposite of reality.
        return in_array($account->type, ['asset', 'expense'])
            ? $debit - $credit
            : $credit - $debit;
    }
}
```

---

## 6. A quick manual test (via Tinker, before there's an API for it)

```powershell
php artisan tinker
```

```php
$entry = JournalEntry::create([
    'reference_number' => 'JE-TEST-001',
    'entry_date' => now()->toDateString(),
    'status' => 'draft',
    'created_by' => 1,
]);
$entry->lines()->create(['account_id' => 1, 'debit' => 100, 'credit' => 0]); // Cash
$entry->lines()->create(['account_id' => 7, 'debit' => 0, 'credit' => 100]); // Sales Revenue

app(App\Services\PostingService::class)->post($entry);
$entry->refresh()->status; // "posted"

app(App\Services\LedgerService::class)->balanceFor(Account::find(1)); // 100.0
```

Try creating an entry with mismatched debit/credit and posting it — you should get the
`UnbalancedJournalEntryException`.

---

## 7. What you built, and why it matters later

- A **single enforcement point** for the core accounting rule — Phase 5's API, Phase 8's
  invoicing, and Phase 19's AI agent all call `PostingService::post()`, never write `status = 'posted'` directly. If you ever need to add a new rule (e.g., "can't post to a closed period"),
  you add it in exactly one place.
- **Reversal instead of edit** as the correction mechanism — the audit trail this produces is what
  makes financial reports (Phase 10) and tax submissions (Phase 15) trustworthy.

## 8. Next: Phase 5

Now that posting is safe, Phase 5 wraps it (and everything else) in a properly designed REST API:
consistent validation, consistent response shapes, and a real error format.
