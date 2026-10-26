# Phase 1 — Environment & Project Bootstrap

**Tags:** `[Backend]` `[Frontend]`
**Depends on:** nothing — this is the starting line.
**You'll finish this phase with:** a Laravel API and a Vue app, both running on your machine,
proven to talk to each other via one real HTTP call.

---

## 1. Learning objectives

By the end of this phase you should be able to explain, in your own words:

- Why we're building a **decoupled** backend (API) and frontend (SPA) instead of one Laravel app
  that renders Blade views.
- What each installed tool actually does (not just "run this command").
- Why `.env` files exist and why they're never committed to Git.
- What CORS is and why the browser blocks your first API call until you configure it.

## 2. Why this phase comes first

You can't write a single line of accounting logic until the two halves of the app can talk to
each other. Every later phase — the ledger, the API design, the offline layer — assumes this
plumbing already works. Get it right once here, and every phase after this is "add a feature,"
not "fight the setup."

---

## 3. Section A — Install the core tools

Given your machine has ~6.5GB free on `C:` and ~52GB free on `D:`, install everything you can
onto `D:`. I'm recommending **Laragon** over Docker/Laravel Sail for this exact reason — Docker
Desktop's WSL2 virtual disk routinely grows to 10–20GB+ on `C:` even for small projects, which
you don't have room for right now. Laragon gives you PHP, MySQL, and Nginx/Apache natively with a
tiny footprint, and you can switch to Docker later if you ever need to (e.g., matching a
production container setup) — nothing here locks you out of that.

### 3.1 Git

Download from https://git-scm.com/download/win and install with defaults.
*Why:* version control from commit #1 — even solo, on a learning project, you want the ability to
undo a bad change.

### 3.2 Laragon (PHP, MySQL, local domains)

1. Download the **Full** edition from https://laragon.org/download/
2. **During install, choose `D:\laragon` as the install path**, not the `C:` default.
3. Laragon bundles PHP — after install, open Laragon → right-click → **PHP** → make sure a
   version **8.3 or newer** is selected (Laravel 13 requires PHP 8.3–8.5).
4. Start Laragon and click **Start All** — this boots Apache/Nginx and MySQL.

*Why Laragon specifically:* it's lightweight, widely used in the MENA PHP community, and gives
you a real MySQL server plus automatic local domains (e.g. `eaccounting.test`) without manually
editing your Windows `hosts` file.

### 3.3 Composer (PHP's package manager)

1. Download from https://getcomposer.org/download/ (Windows installer).
2. During install, it will detect the PHP from Laragon — point it at
   `D:\laragon\bin\php\php-8.3.x\php.exe` if it doesn't find it automatically.

*Why:* Laravel itself, and every package we add (Sanctum, spatie/permission, mPDF...) is
installed via Composer. It's PHP's equivalent of npm.

### 3.4 Node.js

Install **Node.js 24 (Active LTS)** from https://nodejs.org — this is the current LTS line as of
2026 and is what the Vue/Vite tooling expects.

*Why a specific LTS version:* Vite and modern npm packages regularly drop support for old Node
versions. Using the active LTS avoids "works on my machine" package-compatibility issues later.

### 3.5 VS Code extensions

You already have VS Code + the Claude Code extension. Add these:

- **PHP Intelephense** — PHP autocomplete/navigation.
- **Laravel Extension Pack** (by amiralizadeh9480, or similar) — Blade/route/artisan helpers.
- **Vue (Official)** — Vue 3 syntax support, replaces the old "Volar" naming.
- **ESLint** and **Prettier** — JS/Vue formatting and linting.

---

## 4. Section B — Verify every install

Open a terminal (PowerShell) and run each of these. I've included what a correct result looks
like so you know what "working" means, not just "no red text."

```powershell
git --version
# git version 2.4x.x.windows.x

php --version
# PHP 8.3.x (cli) ...  <- must be 8.3 or higher

composer --version
# Composer version 2.x.x

node --version
# v24.x.x

npm --version
# 1x.x.x
```

If `php` or `composer` aren't recognized, it's almost always a PATH issue — Laragon usually adds
itself to PATH automatically; if not, add `D:\laragon\bin\php\php-8.3.x` to your Windows PATH
environment variable and open a **new** terminal.

---

## 5. Section C — Point tool caches at `D:` (protecting your `C:` space)

Composer and npm both cache downloaded packages, and that cache grows over time. Redirect both to
`D:` now, before you start installing dependencies:

```powershell
# Composer's global cache/config directory
setx COMPOSER_HOME "D:\dev-cache\composer"

# npm's cache directory
npm config set cache "D:\dev-cache\npm-cache" --global
```

Close and reopen your terminal after `setx` (it doesn't apply to the current session).

*Principle:* this is a small thing now, but it's the same instinct as designing the database
schema well before data piles up — fix the thing that gets worse over time, early.

---

## 6. Section D — The project folder structure

We'll use one Git repository containing both projects, side by side:

```
eaccounting/
├── backend/          ← Laravel API project (Phase 1 creates this)
├── frontend/          ← Vue SPA project (Phase 1 creates this)
├── docs/              ← this curriculum, lives with the code
└── .gitignore
```

*Why one repo (a "monorepo") instead of two:* while learning, you want to open one VS Code
window and see backend and frontend at once, and commit a backend change + the frontend change
it required in a single commit. If you later need independent deploy pipelines or separate
access control per repo, splitting a monorepo is a simple `git filter-repo` operation — starting
split and merging later is much more painful. Choose the easy-to-reverse option now.

Create the parent folder wherever you keep projects, e.g.:

```powershell
mkdir D:\projects\eaccounting
cd D:\projects\eaccounting
```

---

## 7. Section E — Bootstrap the Laravel backend

### 7.1 Create the project

```powershell
composer create-project laravel/laravel backend "13.*"
cd backend
```

*What this does:* Composer downloads the Laravel 13 skeleton — the framework, its dependencies,
and a standard folder structure (`app/`, `routes/`, `database/`, `config/`, ...) — into `backend/`.

### 7.2 Understand `.env` before touching it

Open `backend/.env`. This file holds every setting that differs between your machine, a
teammate's machine, and production — database credentials, app URL, mail settings, API keys.

**Why it's never committed to Git:** it contains secrets (DB password, later: tax-authority API
keys, signing certificates). Laravel ships a `.env.example` (safe to commit, no real secrets) and
generates your real `.env` locally. Confirm `.env` is in `.gitignore` — Laravel includes this by
default, but it's worth checking now since it matters for every phase after this.

### 7.3 Create the database

Open Laragon → click the **Database** icon (opens HeidiSQL) → create a new database named
`eaccounting`. (We'll design its actual tables in Phase 2 — for now we just need an empty
database Laravel can connect to.)

### 7.4 Configure `.env` to match

```env
APP_NAME="E-Accounting"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=eaccounting
DB_USERNAME=root
DB_PASSWORD=
```

*Security note, now, so it's a habit from day one:* `APP_DEBUG=true` is fine locally — it shows
full error traces to help you debug. In production this **must** be `false`, or you'll leak stack
traces (including file paths and query details) to anyone who triggers an error. We'll enforce
this properly in Phase 20, but the habit starts here.

### 7.5 Run the initial migrations and start the server

```powershell
php artisan migrate
php artisan serve
```

Visit `http://127.0.0.1:8000` — you should see the Laravel welcome page. That confirms: PHP runs,
Composer's dependencies are installed, the app can reach MySQL, and the built-in dev server works.

### 7.6 Create your first real endpoint: a health check

We're not writing accounting logic yet — we're proving the wire between frontend and backend
works, end to end, with the simplest possible payload.

Open `backend/routes/api.php` and add:

```php
<?php

use Illuminate\Support\Facades\Route;

// A closure route is fine for a trivial check like this.
// From Phase 3 onward, real endpoints go through Controllers — not closures —
// so that validation, authorization, and testing all have a proper home.
Route::get('/health', function () {
    return response()->json([
        'status'  => 'ok',
        'service' => 'eaccounting-api',
        'time'    => now()->toIso8601String(),
    ]);
});
```

*Why `routes/api.php` and not `routes/web.php`:* `api.php` routes are automatically prefixed with
`/api` and, critically, are **stateless** — no session cookies, no CSRF token expected. That's the
correct model for a decoupled SPA talking over tokens (Sanctum, from Phase 3 onward).

Restart `php artisan serve` if it's running, then check in a browser or with curl:

```powershell
curl http://127.0.0.1:8000/api/health
# {"status":"ok","service":"eaccounting-api","time":"2026-..."}
```

---

## 8. Section F — Bootstrap the Vue frontend

Open a **new** terminal (keep `php artisan serve` running in the first one).

```powershell
cd D:\projects\eaccounting
npm create vite@latest frontend -- --template vue
cd frontend
npm install
```

### 8.1 Add the libraries this project needs from day one

```powershell
npm install vue-router pinia axios
```

- **vue-router** — page navigation (Dashboard, Chart of Accounts, Invoices, ...).
- **pinia** — shared app state (logged-in user, later: the offline sync queue).
- **axios** — HTTP client for calling the Laravel API.

### 8.2 Folder structure

Inside `frontend/src/`, create:

```
src/
├── views/          ← one file per page/route
├── stores/         ← Pinia stores
├── router/          ← Vue Router config
├── services/        ← API client code (axios wrappers) lives here, never inline in components
└── components/      ← reusable UI pieces
```

*Principle:* components should never call `axios` directly. Every API call goes through
`services/`, so that when we add offline support in Phase 13, there's exactly one place to
intercept "am I online?" logic — not fifty components each making their own decision.

### 8.3 The API client

`src/services/api.js`:

```javascript
import axios from 'axios'

// One shared axios instance, configured once.
// Every service file (authService.js, ledgerService.js, ...) imports THIS,
// rather than creating its own axios instance — so base URL, headers,
// and (later) auth tokens are set in exactly one place.
const api = axios.create({
  baseURL: 'http://127.0.0.1:8000/api',
  headers: {
    'Accept': 'application/json',
  },
})

export default api
```

`src/services/healthService.js`:

```javascript
import api from './api'

// Each "service" file wraps the calls for one backend concern.
// This is the whole file for now — it grows as the backend grows.
export function checkHealth() {
  return api.get('/health')
}
```

### 8.4 Prove the connection from the UI

Replace the contents of `src/App.vue` with:

```vue
<script setup>
// Composition API: `ref` gives us reactive state without a separate
// `data()` function — this is the modern Vue 3 style we'll use throughout.
import { ref, onMounted } from 'vue'
import { checkHealth } from './services/healthService'

const status = ref('checking...')

onMounted(async () => {
  try {
    const response = await checkHealth()
    status.value = `Backend says: ${response.data.status} (at ${response.data.time})`
  } catch (error) {
    status.value = 'Could not reach backend — is `php artisan serve` running?'
  }
})
</script>

<template>
  <main>
    <h1>E-Accounting — Phase 1 Check</h1>
    <p>{{ status }}</p>
  </main>
</template>
```

### 8.5 Run it

```powershell
npm run dev
```

Open the URL Vite prints (usually `http://localhost:5173`). You should see the health check
message — but first you'll likely hit a **CORS error** in the browser console. That's expected;
fixing it is the last real step of this phase, and it's worth understanding *why* it happens.

---

## 9. Section G — CORS: why the browser blocks this, and how to fix it

**What's happening:** your Vue app runs on `http://localhost:5173`. Your API runs on
`http://127.0.0.1:8000`. To the browser, those are two different "origins" (different port =
different origin), and browsers block JavaScript from one origin reading responses from another
unless the server explicitly allows it. This is a **browser security feature**, not a bug — it's
what stops a malicious site from silently calling your bank's API using your logged-in session.

**The fix**, in `backend/config/cors.php`:

```php
'paths' => ['api/*'],

'allowed_methods' => ['*'],

// Only allow our known frontend dev server for now.
// This list grows later to include your production frontend domain —
// it should NEVER be '*' once Sanctum cookies/tokens are in play.
'allowed_origins' => ['http://localhost:5173'],

'allowed_headers' => ['*'],

'supports_credentials' => true,
```

Restart `php artisan serve`, refresh the Vue app — the health check message should now appear.

---

## 10. Section H — Git

Back at the repo root:

```powershell
cd D:\projects\eaccounting
git init
```

Create `.gitignore` at the root:

```gitignore
# Backend
backend/vendor/
backend/.env
backend/storage/*.key
backend/node_modules/

# Frontend
frontend/node_modules/
frontend/dist/
frontend/.env

# OS
.DS_Store
Thumbs.db
```

```powershell
git add .
git commit -m "Phase 1: Laravel API + Vue SPA bootstrap, health check working end to end"
```

---

## 11. What you actually built, and why it matters later

- **A decoupled architecture** — the API knows nothing about how it's consumed (browser, mobile
  app, or later, a service worker replaying queued offline requests). This is what makes Phase 12
  (PWA) and Phase 13 (offline sync) possible without rearchitecting.
- **A services layer on the frontend** — one seam where "online vs offline" logic will live,
  instead of being scattered through every component.
- **The `.env` / secrets discipline**, and the `APP_DEBUG` habit — small now, mandatory once real
  tax-authority credentials and signing keys enter the picture in Phase 15.
- **CORS configured deliberately**, not with a wildcard — the first of many small security
  decisions that compound.

---

## 12. Common Windows issues

| Symptom                   | Likely cause                                    | Fix                                                                            |
| ------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------ |
| `php` not recognized    | PATH not updated, or old terminal               | Open a new terminal; check PATH includes Laragon's PHP bin folder              |
| Port 8000 already in use  | Something else using it                         | `php artisan serve --port=8001` and update the frontend `baseURL` to match |
| MySQL connection refused  | Laragon services not started                    | Open Laragon, click**Start All**                                         |
| `npm install` very slow | Antivirus scanning`node_modules` in real time | Add an exclusion for your project folder in Windows Defender                   |

---

## 13. Next: Phase 2

Once `php artisan serve` and `npm run dev` are both running and the health check shows up in the
browser, you're done with this phase. Phase 2 designs the actual database schema — Chart of
Accounts, Journal Entries, the Ledger — the foundation everything else in the app is built on.
Let me know when you're ready and we'll build it together.
