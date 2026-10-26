# Phase 11 — Multi-Tenancy: Evolving Single→Multi

**Tags:** `[Backend]` `[Security]`
**Depends on:** everything through Phase 10 (there's now a real single-tenant app to evolve)
**You'll finish this phase with:** the exact same app, now capable of serving many separate
businesses from one deployment, with one business's data provably invisible to another's.

---

## 1. Learning objectives

- Understand the two mainstream multi-tenancy strategies and **why we're choosing one deliberately
  over the other** for this project, not by default.
- Practice Laravel's **Global Scopes** — the single mechanism that makes "every query is
  automatically tenant-filtered" possible without repeating a `where` clause everywhere.
- Understand why multi-tenancy is a **security** feature as much as a data-modeling one: a bug
  here means Business A can see Business B's ledger.

## 2. The two approaches, and which we're using

| Approach                                                      | How it works                                                                              | Trade-off                                                                                                                                                                |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Shared schema, `tenant_id` column** (what we build) | One database, every business table gets a`tenant_id` column, every query filtered by it | Simple to run and back up; easy to add later on an existing app (this phase); isolation depends entirely on code discipline (a missing scope = a data leak)              |
| **Database-per-tenant**                                 | Each business gets its own database, connection switched per request                      | Strongest isolation (a bug can't leak across databases); harder migrations (run every migration N times), harder to scale to thousands of tiny tenants, harder local dev |

We use shared-schema because: (a) it's the realistic evolution of what you already built, rather
than a rewrite, and (b) at the scale a learning project (and most real SMB SaaS products) starts
at, the operational simplicity outweighs the isolation upside. If this app ever needs to serve a
handful of very large enterprise clients who demand physical data separation, database-per-tenant
for *those specific* clients is a targeted addition later — not a wholesale rewrite.

---

## 3. Schema changes

```powershell
php artisan make:migration create_tenants_table
php artisan make:migration add_tenant_id_to_business_tables
```

```php
// tenants
Schema::create('tenants', function (Blueprint $table) {
    $table->id();
    $table->string('name'); // business name
    $table->string('slug')->unique(); // used in subdomains later, e.g. acme.eaccounting.app
    $table->timestamps();
});
```

```php
// add_tenant_id_to_business_tables — every table holding business data
foreach (['accounts', 'contacts', 'journal_entries', 'invoices', 'payments'] as $tableName) {
    Schema::table($tableName, function (Blueprint $table) {
        $table->foreignId('tenant_id')->after('id')->constrained();
    });
}
```

Add `tenant_id` to `users` too (each user belongs to exactly one tenant, for now — a user
belonging to multiple businesses is a real feature some SaaS apps need, but it's a deliberate
scope cut here to keep this phase learnable).

*Why `accounts.code` was `unique()` in Phase 2, and why that has to change now:* two different
businesses will both want an account coded `"1000"`. Update that migration/constraint to
`unique(['tenant_id', 'code'])` — a good concrete example of a decision from Phase 2 that
**had** to be revisited once multi-tenancy was added; this is exactly why Phase 2 didn't hardcode
anything into a raw unique index without knowing this phase was coming.

---

## 4. The Global Scope — automatic filtering

```powershell
php artisan make:scope TenantScope
```

```php
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if ($tenantId = app('currentTenantId')) {
            $builder->where($model->getTable() . '.tenant_id', $tenantId);
        }
    }
}
```

Apply it via a shared trait rather than repeating `static::addGlobalScope(...)` in five models:

```php
trait BelongsToTenant
{
    protected static function bootBelongsToTenant(): void
    {
        static::addGlobalScope(new TenantScope);

        // Auto-fill tenant_id on create, so every service from Phase 4/8 onward
        // (PostingService, InvoiceService...) needs ZERO changes — they call
        // Account::create()/JournalEntry::create() exactly as before, and the
        // correct tenant_id is attached automatically.
        static::creating(function ($model) {
            if (! $model->tenant_id && app()->bound('currentTenantId')) {
                $model->tenant_id = app('currentTenantId');
            }
        });
    }
}
```

Add `use BelongsToTenant;` to `Account`, `Contact`, `JournalEntry`, `Invoice`, `Payment`.

*This is the payoff of Phases 2–10's discipline:* because every write went through a service
(`PostingService::post()`, `InvoiceService::create()`), not scattered raw `DB::table()->insert()`
calls, adding tenant isolation here required touching **one trait and one line per model** — not
rewriting business logic.

---

## 5. Identifying the tenant per request

`app/Http/Middleware/IdentifyTenant.php`:

```php
class IdentifyTenant
{
    public function handle(Request $request, Closure $next)
    {
        $user = $request->user(); // set by Sanctum's auth:sanctum middleware, which runs first

        if ($user) {
            app()->instance('currentTenantId', $user->tenant_id);
        }

        return $next($request);
    }
}
```

Register it to run **after** `auth:sanctum` in `bootstrap/app.php`'s middleware config, on the
same `v1` group. Every controller, service, and query written in Phases 4–10 now automatically
sees only its tenant's data — none of that code needs to change.

*Security-critical detail:* `TenantScope` degrades to "no filter" if `currentTenantId` was never
set — which would happen for, say, a queued job running outside an HTTP request context (Phase
18's scheduled jobs). Every job that touches tenant data must explicitly set the tenant context
itself (`app()->instance('currentTenantId', $job->tenantId)`) before doing any queries — this is
flagged again in Phase 18/20 because it's the most common real-world tenancy bug.

---

## 6. Registration now creates a tenant

Update Phase 3's `AuthController::register()`:

```php
public function register(RegisterRequest $request)
{
    return DB::transaction(function () use ($request) {
        $tenant = Tenant::create([
            'name' => $request->business_name,
            'slug' => Str::slug($request->business_name) . '-' . Str::random(4),
        ]);

        $user = User::create([
            'tenant_id' => $tenant->id,
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);
        $user->assignRole('admin'); // the first user of a new tenant is its admin

        app()->instance('currentTenantId', $tenant->id);
        app(ChartOfAccountsSeeder::class)->run(); // every new tenant gets a starter chart of accounts

        $token = $user->createToken('spa')->plainTextToken;
        return response()->json(['user' => $user, 'tenant' => $tenant, 'token' => $token], 201);
    });
}
```

---

## 7. What you built, and why it matters later

- **One codebase, safely serving many businesses** — the AI assistant (Phase 19), the e-commerce
  connector (Phase 17), and the tax integration (Phase 15) all now need to be tenant-aware in
  exactly one way: never do work outside a request without explicitly setting
  `currentTenantId` first.
- Confirmed, in practice, that routing every write through a service (the discipline since Phase
  4) is what made this a **small, contained** change instead of a rewrite.

## 8. Next: Phase 12

Time to make the app installable — Phase 12 turns the Vue SPA into a real PWA: a manifest, a
service worker, and "Add to desktop," while still assuming a mostly-online connection.
