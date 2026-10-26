# Phase 3 — Authentication & Authorization

**Tags:** `[Backend]` `[API]` `[Security]`
**Depends on:** Phase 2 (schema exists, `users` table already ships with Laravel)
**You'll finish this phase with:** register/login/logout working end to end from the Vue app,
tokens attached automatically to every API call, and two roles (`admin`, `accountant`) restricting
what each user can do.

---

## 1. Learning objectives

- Understand why a stateless, decoupled SPA uses **Bearer tokens**, not PHP sessions.
- Understand the difference between **authentication** ("who are you") and **authorization**
  ("what are you allowed to do") — and why they're separate systems (Sanctum vs. permissions).
- Know the real security trade-off of storing an API token in the browser, and what we're
  deferring to Phase 20 to improve.

## 2. Why this phase now

`journal_entries.created_by` (Phase 2) is meaningless without real authenticated users, and every
API endpoint from here on needs to know who's calling and whether they're allowed to.

---

## 3. Install the packages

```powershell
cd backend
composer require laravel/sanctum
composer require spatie/laravel-permission

php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

*Why two separate packages instead of one "auth" solution:* Sanctum answers "is this a real,
logged-in user" (authentication). spatie/laravel-permission answers "is this user an admin"
(authorization). Keeping them separate means you can change how someone logs in later (e.g. add
Google SSO) without touching a single line of role/permission logic.

---

## 4. Wire up the User model

`app/Models/User.php`:

```php
use Laravel\Sanctum\HasApiTokens;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasApiTokens, HasRoles;
    // ...existing code
}
```

---

## 5. Seed roles

```powershell
php artisan make:seeder RolesSeeder
```

```php
class RolesSeeder extends Seeder
{
    public function run(): void
    {
        Role::create(['name' => 'admin']);
        Role::create(['name' => 'accountant']);
        Role::create(['name' => 'viewer']);
    }
}
```

*Principle:* start with the roles you actually need (admin = full access, accountant = can post/
void, viewer = read-only reports) rather than a fine-grained permission matrix you don't need
yet. spatie/laravel-permission supports per-permission grants too — add those when a real need
appears, not speculatively.

---

## 6. The Auth API

`app/Http/Controllers/Api/AuthController.php`:

```php
class AuthController extends Controller
{
    public function register(RegisterRequest $request)
    {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);
        $user->assignRole('viewer'); // safest default — an admin promotes them later

        $token = $user->createToken('spa')->plainTextToken;

        return response()->json(['user' => $user, 'token' => $token], 201);
    }

    public function login(LoginRequest $request)
    {
        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            // Same error for "no such user" and "wrong password" — never reveal
            // which one it was, or you've handed an attacker a user-enumeration tool.
            throw ValidationException::withMessages([
                'email' => ['These credentials do not match our records.'],
            ]);
        }

        $token = $user->createToken('spa')->plainTextToken;

        return response()->json(['user' => $user, 'token' => $token]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();
        return response()->json(['message' => 'Logged out']);
    }

    public function me(Request $request)
    {
        return response()->json($request->user()->load('roles'));
    }
}
```

`app/Http/Requests/RegisterRequest.php` and `LoginRequest.php` — Form Requests, not inline
validation, so the rule lives in one reusable, testable place:

```php
class RegisterRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', 'string', 'min:8', 'confirmed'],
        ];
    }
}
```

`routes/api.php`:

```php
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/me', [AuthController::class, 'me']);
});
```

*Why `middleware('auth:sanctum')` on a group, not each route:* as we add dozens of accounting
endpoints in later phases, every single one goes inside this group. Repeating the middleware
per-route is how "oops, I forgot to protect one endpoint" security bugs happen.

---

## 7. Restricting an action by role — a real example

Only `admin` and `accountant` should be able to post a journal entry (from Phase 4 onward). In the
controller method:

```php
public function post(JournalEntry $journalEntry)
{
    if (! auth()->user()->hasAnyRole(['admin', 'accountant'])) {
        abort(403, 'You do not have permission to post journal entries.');
    }
    // ...posting logic (Phase 4)
}
```

For anything checked in more than one place, extract it into a **Policy**
(`php artisan make:policy JournalEntryPolicy --model=JournalEntry`) instead of repeating the
`hasAnyRole` check — Phase 5 uses policies consistently once there are real endpoints to protect.

---

## 8. Frontend: the auth store and automatic token attachment

`frontend/src/stores/authStore.js`:

```javascript
import { defineStore } from 'pinia'
import api from '../services/api'

export const useAuthStore = defineStore('auth', {
  state: () => ({
    user: null,
    // Known trade-off, revisited in Phase 20: localStorage is readable by any JS
    // running on the page, so an XSS bug elsewhere in the app could steal this token.
    // httpOnly cookies avoid that but need same-site cookie handling, which is more
    // setup than a learning project needs on day one. We accept the risk now and
    // harden it later once there's a real app to harden.
    token: localStorage.getItem('token') || null,
  }),
  actions: {
    async login(email, password) {
      const { data } = await api.post('/login', { email, password })
      this.user = data.user
      this.token = data.token
      localStorage.setItem('token', data.token)
    },
    async logout() {
      await api.post('/logout')
      this.user = null
      this.token = null
      localStorage.removeItem('token')
    },
    async fetchMe() {
      const { data } = await api.get('/me')
      this.user = data
    },
  },
})
```

Update `frontend/src/services/api.js` to attach the token to every request automatically:

```javascript
import axios from 'axios'

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://127.0.0.1:8000/api',
  headers: { Accept: 'application/json' },
})

// Runs before every request — one place to attach auth, instead of every
// service file remembering to add the header itself.
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

export default api
```

(Note the switch to `import.meta.env.VITE_API_BASE_URL` — add `VITE_API_BASE_URL=http://127.0.0.1:8000/api`
to a new `frontend/.env` file. Hardcoding the URL was fine for Phase 1's one-off check; from here
on, config belongs in `.env`, same principle as the backend.)

### Router guard

`frontend/src/router/index.js`:

```javascript
router.beforeEach((to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.token) {
    return { name: 'login' }
  }
})
```

---

## 9. What you built, and why it matters later

- Every future endpoint just adds itself to the `auth:sanctum` group — the security boundary is
  centralized, not repeated.
- Roles exist as data (`admin`/`accountant`/`viewer`), not hardcoded `if ($user->email === '...')`
  checks — this is what lets the AI agent permission model in Phase 19 hook into the same system
  instead of inventing a second one.
- The token-storage trade-off is written down now, not discovered as a surprise in a security
  review later.

## 10. Next: Phase 4

With real, authenticated, role-checked users, Phase 4 builds the actual double-entry posting
engine — the code that turns a draft journal entry into an immutable, balanced, posted one.
