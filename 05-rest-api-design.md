# Phase 5 — REST API Design

**Tags:** `[API]` `[Security]`
**Depends on:** Phase 4 (something worth exposing)
**You'll finish this phase with:** a versioned, validated, consistently-shaped API for journal
entries — the template every later endpoint (invoices, reports, tax submissions...) copies.

---

## 1. Learning objectives

- Understand why APIs are **versioned** from day one, even with one consumer (your own frontend).
- Understand the separation: **Form Requests** validate input, **Policies** authorize the action,
  **API Resources** shape the output — three different jobs, three different classes.
- Build one **consistent error format** the frontend can rely on everywhere.

## 2. Why this phase now

Phase 4 proved the accounting logic works from Tinker. Real users (and the AI agent, later) need
it over HTTP, with input they don't control validated, and errors shaped predictably enough that
the frontend can show a sensible message instead of a raw stack trace.

---

## 3. Versioning: group everything under `/api/v1`

`routes/api.php`:

```php
Route::prefix('v1')->group(function () {
    Route::post('/register', [AuthController::class, 'register']);
    Route::post('/login', [AuthController::class, 'login']);

    Route::middleware('auth:sanctum')->group(function () {
        Route::post('/logout', [AuthController::class, 'logout']);
        Route::get('/me', [AuthController::class, 'me']);

        Route::apiResource('journal-entries', JournalEntryController::class);
        Route::post('/journal-entries/{journalEntry}/post', [JournalEntryController::class, 'post']);
        Route::post('/journal-entries/{journalEntry}/reverse', [JournalEntryController::class, 'reverse']);
    });
});
```

Update `frontend/.env`: `VITE_API_BASE_URL=http://127.0.0.1:8000/api/v1`.

*Why version now, with one consumer:* the day you need to change a response shape for the mobile
app (or a public integration API in Phase 17), you'll be glad old clients can keep hitting `/v1`
while `/v2` rolls out. Adding versioning to an already-live unversioned API is a painful migration
that this five-minute decision avoids entirely.

---

## 4. Form Requests: validate, don't trust

```powershell
php artisan make:request StoreJournalEntryRequest
```

```php
class StoreJournalEntryRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', JournalEntry::class);
    }

    public function rules(): array
    {
        return [
            'entry_date' => ['required', 'date'],
            'description' => ['nullable', 'string', 'max:255'],
            'lines' => ['required', 'array', 'min:2'],
            'lines.*.account_id' => ['required', 'exists:accounts,id'],
            'lines.*.debit' => ['required', 'numeric', 'min:0'],
            'lines.*.credit' => ['required', 'numeric', 'min:0'],
        ];
    }

    public function withValidator($validator): void
    {
        // Cross-field rules (the balance check) don't fit Laravel's per-field
        // rule syntax, so they go here — this is the FIRST line of defense.
        // PostingService (Phase 4) is the SECOND, unconditional one — a Form
        // Request can be bypassed by a direct service call from a job or the
        // AI agent, PostingService cannot.
        $validator->after(function ($validator) {
            $lines = collect($this->input('lines', []));
            if (bccomp((string) $lines->sum('debit'), (string) $lines->sum('credit'), 2) !== 0) {
                $validator->errors()->add('lines', 'Debits and credits must balance.');
            }
        });
    }
}
```

---

## 5. Policies: authorize, separately from validation

```powershell
php artisan make:policy JournalEntryPolicy --model=JournalEntry
```

```php
class JournalEntryPolicy
{
    public function create(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'accountant']);
    }

    public function post(User $user, JournalEntry $entry): bool
    {
        return $user->hasAnyRole(['admin', 'accountant']);
    }

    public function view(User $user, JournalEntry $entry): bool
    {
        return true; // all authenticated roles, including viewer, can read
    }
}
```

Register it in `app/Providers/AppServiceProvider.php`'s `boot()`:

```php
Gate::policy(JournalEntry::class, JournalEntryPolicy::class);
```

---

## 6. API Resources: shape the output once

```powershell
php artisan make:resource JournalEntryResource
php artisan make:resource JournalEntryLineResource
```

```php
class JournalEntryResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'reference_number' => $this->reference_number,
            'entry_date' => $this->entry_date->toDateString(),
            'description' => $this->description,
            'status' => $this->status,
            'lines' => JournalEntryLineResource::collection($this->whenLoaded('lines')),
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

*Why a Resource instead of returning the Eloquent model directly:* `return $journalEntry;` leaks
every raw column (including ones you might add later that shouldn't be public, like an internal
sync flag from Phase 13) and ties your API's shape directly to your database schema. A Resource is
one deliberate place that decides "this is what the outside world sees."

---

## 7. The controller, tying it together

```php
class JournalEntryController extends Controller
{
    public function __construct(private PostingService $postingService) {}

    public function index()
    {
        $entries = JournalEntry::with('lines')->latest('entry_date')->paginate(25);
        return JournalEntryResource::collection($entries);
    }

    public function store(StoreJournalEntryRequest $request)
    {
        $entry = DB::transaction(function () use ($request) {
            $entry = JournalEntry::create([
                'reference_number' => 'JE-' . now()->format('Y') . '-' . str_pad(JournalEntry::count() + 1, 6, '0', STR_PAD_LEFT),
                'entry_date' => $request->entry_date,
                'description' => $request->description,
                'status' => 'draft',
                'created_by' => $request->user()->id,
            ]);
            $entry->lines()->createMany($request->lines);
            return $entry;
        });

        return new JournalEntryResource($entry->load('lines'));
    }

    public function post(JournalEntry $journalEntry)
    {
        $this->authorize('post', $journalEntry);
        $posted = $this->postingService->post($journalEntry);
        return new JournalEntryResource($posted->load('lines'));
    }

    public function reverse(JournalEntry $journalEntry, Request $request)
    {
        $this->authorize('post', $journalEntry);
        $reversal = $this->postingService->reverse($journalEntry, $request->input('reason', 'No reason given'));
        return new JournalEntryResource($reversal->load('lines'));
    }
}
```

---

## 8. One consistent error shape, everywhere

In `bootstrap/app.php` (Laravel 11+/13's exception configuration):

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (Throwable $e, Request $request) {
        if (! $request->is('api/*')) {
            return null; // let Laravel handle web routes normally
        }

        $status = match (true) {
            $e instanceof ValidationException => 422,
            $e instanceof AuthenticationException => 401,
            $e instanceof AuthorizationException => 403,
            $e instanceof ModelNotFoundException => 404,
            $e instanceof UnbalancedJournalEntryException => 422,
            default => 500,
        };

        return response()->json([
            'error' => [
                'message' => $e->getMessage(),
                // Never leak a stack trace to the client — this is exactly
                // what APP_DEBUG=false (Phase 1) exists to prevent in production.
                'details' => $e instanceof ValidationException ? $e->errors() : null,
            ],
        ], $status);
    });
})
```

Now every failure — validation, auth, a business-rule exception — arrives at the frontend as
`{ error: { message, details } }`, so the frontend's error handling (next section) is written
once, not per-endpoint.

---

## 9. Rate limiting, from day one

`routes/api.php`, wrap the whole `v1` group:

```php
Route::prefix('v1')->middleware('throttle:api')->group(function () {
    // ... all routes
});
```

`config/rate_limits` (or `RouteServiceProvider`/`bootstrap/app.php` depending on your Laravel
version) — define `api` as e.g. 120 requests/minute per user, tighter (e.g. 5/minute) specifically
on `/login` to blunt brute-force attempts:

```php
RateLimiter::for('login', fn (Request $request) => Limit::perMinute(5)->by($request->ip()));
```

---

## 10. Frontend: handle the standard error shape once

`frontend/src/services/api.js`, extend the interceptor:

```javascript
api.interceptors.response.use(
  (response) => response,
  (error) => {
    const message = error.response?.data?.error?.message || 'Something went wrong.'
    const details = error.response?.data?.error?.details || null
    return Promise.reject({ message, details, status: error.response?.status })
  }
)
```

Every component that calls the API now catches one predictable shape, instead of parsing axios
errors individually.

---

## 11. What you built, and why it matters later

- A **template**: Form Request → Policy → Service → Resource. Every domain object added from
  Phase 8 onward (invoices, payments, cheques) follows this exact shape.
- A single error contract the frontend, and later the offline sync queue (Phase 13), can rely on.
- Versioning and rate limiting in place before there's any real traffic to retrofit them under.

## 12. Next: Phase 6

Time to give this API a real interface — Phase 6 builds out the Vue app's structural foundation
(stores, router, layout) so Phase 7 can build actual accounting screens on top of it.
