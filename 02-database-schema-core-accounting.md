* [ ] 

# Phase 2 — Database Design: Core Accounting Schema (Single-Tenant)

**Tags:** `[Backend]`
**Depends on:** Phase 1 (Laravel + Vue running)
**You'll finish this phase with:** the four tables every accounting feature in this app sits on —
Chart of Accounts, Contacts, Journal Entries, and Journal Entry Lines — plus a seeded starter
chart of accounts.

---

## 1. Learning objectives

- Understand **why double-entry bookkeeping needs two tables** (a header + lines), not one.
- Understand why money columns are `decimal`, never `float`.
- Understand why accounting tables use soft deletes and are treated as append-only/immutable
  once posted (enforced properly in Phase 4 — here we just shape the schema for it).

## 2. Why this phase now

Everything else — auth, the posting engine, the API, every report — reads or writes these four
tables. Get the shape right now; changing it later means writing data migrations against real
transactions, which is exactly the kind of risky change accounting software should avoid.

---

## 3. The core principle: double-entry as two tables

Every financial event (a sale, a payment, a bill) is recorded as **one journal entry** made of
**two or more lines**, where the lines' debits and credits balance to zero. Example — a $100 cash
sale:

| Line | Account       | Debit  | Credit |
| ---- | ------------- | ------ | ------ |
| 1    | Cash          | 100.00 |        |
| 2    | Sales Revenue |        | 100.00 |

That's why we need a `journal_entries` **header** table (one row per event: date, reference,
status) and a `journal_entry_lines` **detail** table (one row per debit or credit within that
event). A single `amount` + `type` column on one flat table would work for simple cases but makes
it much harder to guarantee "this event's lines balance" as a single, checkable unit — which is
the whole point of double-entry.

---

## 4. Migrations

Run:

```powershell
cd backend
php artisan make:migration create_accounts_table
php artisan make:migration create_contacts_table
php artisan make:migration create_journal_entries_table
php artisan make:migration create_journal_entry_lines_table
```

### 4.1 `accounts` — the Chart of Accounts

```php
public function up(): void
{
    Schema::create('accounts', function (Blueprint $table) {
        $table->id();
        $table->string('code')->unique();   // e.g. "1000" — stable identifier, never reused
        $table->string('name');              // e.g. "Cash on Hand"
        // The five fundamental account types — every account is exactly one of these.
        // This single column is what lets us compute the Balance Sheet vs Income
        // Statement later just by grouping on `type`.
        $table->enum('type', ['asset', 'liability', 'equity', 'revenue', 'expense']);
        // Self-referencing parent lets you build "1000 Assets > 1010 Current Assets > 1011 Cash"
        // hierarchies without a separate tree table.
        $table->foreignId('parent_id')->nullable()->constrained('accounts')->nullOnDelete();
        $table->boolean('is_active')->default(true); // retire an account without deleting history
        $table->timestamps();
        $table->softDeletes(); // never hard-delete an account once it may have transactions
    });
}
```

### 4.2 `contacts` — customers and suppliers

```php
public function up(): void
{
    Schema::create('contacts', function (Blueprint $table) {
        $table->id();
        $table->enum('type', ['customer', 'supplier', 'both']);
        $table->string('name');
        $table->string('tax_number')->nullable();   // needed later for Egypt e-Invoice (Phase 15)
        $table->string('email')->nullable();
        $table->string('phone')->nullable();
        $table->text('address')->nullable();
        $table->timestamps();
        $table->softDeletes();
    });
}
```

### 4.3 `journal_entries` — the header

```php
public function up(): void
{
    Schema::create('journal_entries', function (Blueprint $table) {
        $table->id();
        $table->string('reference_number')->unique(); // human-facing, e.g. "JE-2026-000042"
        $table->date('entry_date');
        $table->string('description')->nullable();
        // draft = editable freely. posted = locked, only a reversing entry can undo it.
        // void = a posted entry that was reversed (kept for audit trail, never deleted).
        $table->enum('status', ['draft', 'posted', 'void'])->default('draft');
        $table->foreignId('created_by')->constrained('users');
        $table->foreignId('reverses_journal_entry_id')->nullable()
            ->constrained('journal_entries')->nullOnDelete();
        $table->timestamps();
        $table->softDeletes();
    });
}
```

### 4.4 `journal_entry_lines` — the detail

```php
public function up(): void
{
    Schema::create('journal_entry_lines', function (Blueprint $table) {
        $table->id();
        $table->foreignId('journal_entry_id')->constrained()->cascadeOnDelete();
        $table->foreignId('account_id')->constrained('accounts');
        $table->foreignId('contact_id')->nullable()->constrained('contacts');
        // decimal(15,2), NEVER float/double — floats can't represent money exactly
        // (0.1 + 0.2 !== 0.3 in binary floating point), which is unacceptable once
        // you're summing thousands of rows into a balance sheet that must foot to zero.
        $table->decimal('debit', 15, 2)->default(0);
        $table->decimal('credit', 15, 2)->default(0);
        $table->string('description')->nullable();
        $table->timestamps();
    });
}
```

Run them:

```powershell
php artisan migrate
```

---

## 5. Eloquent models

```powershell
php artisan make:model Account
php artisan make:model Contact
php artisan make:model JournalEntry
php artisan make:model JournalEntryLine
```

`app/Models/Account.php`:

```php
class Account extends Model
{
    use SoftDeletes;

    protected $fillable = ['code', 'name', 'type', 'parent_id', 'is_active'];

    public function parent(): BelongsTo
    {
        return $this->belongsTo(Account::class, 'parent_id');
    }

    public function children(): HasMany
    {
        return $this->hasMany(Account::class, 'parent_id');
    }

    public function lines(): HasMany
    {
        return $this->hasMany(JournalEntryLine::class);
    }
}
```

`app/Models/JournalEntry.php`:

```php
class JournalEntry extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'reference_number', 'entry_date', 'description', 'status', 'created_by',
    ];

    public function lines(): HasMany
    {
        return $this->hasMany(JournalEntryLine::class);
    }

    public function creator(): BelongsTo
    {
        return $this->belongsTo(User::class, 'created_by');
    }
}
```

`app/Models/JournalEntryLine.php`:

```php
class JournalEntryLine extends Model
{
    protected $fillable = ['journal_entry_id', 'account_id', 'contact_id', 'debit', 'credit', 'description'];

    public function journalEntry(): BelongsTo
    {
        return $this->belongsTo(JournalEntry::class);
    }

    public function account(): BelongsTo
    {
        return $this->belongsTo(Account::class);
    }
}
```

(`Contact.php` is a plain model with `use SoftDeletes;` and the matching `$fillable` — no
relationships needed yet; Phase 8 adds `invoices()`.)

---

## 6. Seed a starter Chart of Accounts

```powershell
php artisan make:seeder ChartOfAccountsSeeder
```

`database/seeders/ChartOfAccountsSeeder.php`:

```php
class ChartOfAccountsSeeder extends Seeder
{
    public function run(): void
    {
        $accounts = [
            ['code' => '1000', 'name' => 'Cash on Hand', 'type' => 'asset'],
            ['code' => '1010', 'name' => 'Bank Account', 'type' => 'asset'],
            ['code' => '1100', 'name' => 'Accounts Receivable', 'type' => 'asset'],
            ['code' => '2000', 'name' => 'Accounts Payable', 'type' => 'liability'],
            ['code' => '2100', 'name' => 'Tax Payable', 'type' => 'liability'],
            ['code' => '3000', 'name' => "Owner's Equity", 'type' => 'equity'],
            ['code' => '4000', 'name' => 'Sales Revenue', 'type' => 'revenue'],
            ['code' => '5000', 'name' => 'Cost of Goods Sold', 'type' => 'expense'],
            ['code' => '5100', 'name' => 'General Expenses', 'type' => 'expense'],
        ];

        foreach ($accounts as $account) {
            Account::create($account);
        }
    }
}
```

Register it in `database/seeders/DatabaseSeeder.php`'s `run()` method with `$this->call(ChartOfAccountsSeeder::class);`, then:

```powershell
php artisan db:seed
```

---

## 7. What you built, and why it matters later

- A schema that **can't easily represent an unbalanced transaction by accident** — the
  header/line split is what Phase 4's `PostingService` will check against.
- `decimal(15,2)` everywhere money appears — a rule you'll repeat in every migration from here on.
- Soft deletes + a `void`/`reverses_journal_entry_id` pattern instead of hard deletes — this *is*
  the audit trail principle that Phase 19's AI agents will depend on to be trustworthy.

## 8. Next: Phase 3

With a schema in place, Phase 3 adds authentication (Sanctum) and roles/permissions
(spatie/laravel-permission) — so that "who is `created_by`" on a journal entry is a real,
authenticated user, and so different roles can be restricted from posting or voiding entries.
