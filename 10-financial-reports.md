# Phase 10 — Financial Reports

**Tags:** `[Backend]` `[Frontend]`
**Depends on:** Phase 4 (ledger), Phase 9 (currency-correct amounts)
**You'll finish this phase with:** Trial Balance, Income Statement (P&L), Balance Sheet, and a
simplified Cash Flow report — all computed live from posted journal entries, never hand-maintained.

---

## 1. Learning objectives

- Understand that in a correctly-built ledger, **every report is a query, not a stored value** —
  if you find yourself wanting a `balance` column that gets manually updated, that's a sign the
  underlying posting logic (Phase 4) isn't being trusted.
- Understand the accounting-identity checks that prove a report is internally consistent
  (Assets = Liabilities + Equity; Trial Balance total debits = total credits).

## 2. Why this phase now

This is the payoff of Phases 2–4: because every transaction was forced through `PostingService`
and double-entry lines, these reports are now genuinely simple — sums and groupings over data you
already trust.

---

## 3. `ReportService`

```php
class ReportService
{
    public function trialBalance(string $asOfDate): array
    {
        $accounts = Account::with(['lines' => function ($q) use ($asOfDate) {
            $q->whereHas('journalEntry', fn ($je) => $je->where('status', 'posted')
                ->where('entry_date', '<=', $asOfDate));
        }])->get();

        $rows = $accounts->map(function ($account) {
            $debit = $account->lines->sum('debit');
            $credit = $account->lines->sum('credit');
            return [
                'account' => "{$account->code} {$account->name}",
                'debit' => $debit > $credit ? $debit - $credit : 0,
                'credit' => $credit > $debit ? $credit - $debit : 0,
            ];
        })->filter(fn ($r) => $r['debit'] > 0 || $r['credit'] > 0);

        return [
            'rows' => $rows->values(),
            'total_debit' => $rows->sum('debit'),
            'total_credit' => $rows->sum('credit'),
            // If this is ever false, something bypassed PostingService — it should be
            // mathematically impossible given Phase 4's guarantee. Treat it as a bug alarm,
            // not a display quirk, if it's ever off.
            'is_balanced' => bccomp((string) $rows->sum('debit'), (string) $rows->sum('credit'), 2) === 0,
        ];
    }

    public function incomeStatement(string $from, string $to): array
    {
        $ledger = app(LedgerService::class);
        $revenue = Account::where('type', 'revenue')->get()
            ->map(fn ($a) => ['name' => $a->name, 'amount' => $ledger->balanceForPeriod($a, $from, $to)]);
        $expenses = Account::where('type', 'expense')->get()
            ->map(fn ($a) => ['name' => $a->name, 'amount' => $ledger->balanceForPeriod($a, $from, $to)]);

        $totalRevenue = $revenue->sum('amount');
        $totalExpenses = $expenses->sum('amount');

        return [
            'revenue' => $revenue, 'total_revenue' => $totalRevenue,
            'expenses' => $expenses, 'total_expenses' => $totalExpenses,
            'net_income' => $totalRevenue - $totalExpenses,
        ];
    }

    public function balanceSheet(string $asOfDate): array
    {
        $ledger = app(LedgerService::class);
        $byType = fn ($type) => Account::where('type', $type)->get()
            ->map(fn ($a) => ['name' => $a->name, 'balance' => $ledger->balanceFor($a, $asOfDate)]);

        $assets = $byType('asset');
        $liabilities = $byType('liability');
        $equity = $byType('equity');

        return [
            'assets' => $assets, 'total_assets' => $assets->sum('balance'),
            'liabilities' => $liabilities, 'total_liabilities' => $liabilities->sum('balance'),
            'equity' => $equity, 'total_equity' => $equity->sum('balance'),
            // The fundamental accounting equation. If this check fails on real data,
            // stop and investigate before trusting anything else the app reports.
            'is_balanced' => bccomp(
                (string) $assets->sum('balance'),
                (string) ($liabilities->sum('balance') + $equity->sum('balance')),
                2
            ) === 0,
        ];
    }
}
```

Add `balanceForPeriod()` to Phase 4's `LedgerService` (same idea as `balanceFor`, but summing only
lines dated between two dates rather than "up to" one date — used for revenue/expense accounts,
which reset conceptually each period, unlike balance sheet accounts which accumulate forever).

*Note on Cash Flow:* a fully correct indirect-method cash flow statement is genuinely complex
(it reconciles net income against non-cash items). Start with a **simplified direct-method**
version for this phase — sum the movements through your Cash and Bank accounts only, bucketed by
whether the offsetting account was revenue/expense (operating), a fixed asset (investing), or
equity/loans (financing). Revisit it properly once Fixed Assets exist as a module, which is
outside this roadmap's initial 21 phases — flag it as a known simplification, not a hidden gap.

---

## 4. API + Frontend

```php
Route::get('/reports/trial-balance', [ReportController::class, 'trialBalance']);
Route::get('/reports/income-statement', [ReportController::class, 'incomeStatement']);
Route::get('/reports/balance-sheet', [ReportController::class, 'balanceSheet']);
```
Each accepts date-range query params and calls the matching `ReportService` method — thin
controllers, all the logic already lives in the service.

`frontend/src/views/ReportsView.vue`: a date-range picker + tabs for each report, each rendering
its rows in a table with a bold total row. Structurally the same "fetch on mount, render a list"
pattern as every earlier screen — the only new piece is the date-range inputs driving the query
params.

---

## 5. What you built, and why it matters later

- Reports that are **provably correct by construction** — the `is_balanced` checks aren't
  decoration, they're a live assertion that Phase 4's invariant is holding across the whole
  system.
- A `ReportService` shape that the AI assistant (Phase 19) can call directly to answer questions
  like "what's our net income this month" — it's already a clean, reusable service, not logic
  buried in a controller or a Blade view.

## 6. Next: Phase 11

The app so far assumes one business. Phase 11 evolves it into a multi-tenant SaaS — the same
codebase, safely serving many businesses at once.
