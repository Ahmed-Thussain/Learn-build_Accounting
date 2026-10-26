# Phase 20 — Security Hardening Pass

**Tags:** `[Backend]` `[Frontend]` `[Security]`
**Depends on:** everything — this is a deliberate review pass, not new features.
**You'll finish this phase with:** 2FA, encryption at rest for sensitive credentials, a
dependency audit, secure headers, and a checklist you re-run before every future major release.

---

## 1. Learning objectives

- Learn to run a **security review as a checklist**, not a vague feeling of "is this secure?"
- Understand **encryption at rest** vs. encryption in transit (HTTPS) — different problems, both
  needed, especially now that Phase 15 stores tax-authority credentials.
- Practice reading a **dependency vulnerability scan** and deciding what actually needs action.

## 2. Why this phase now, as a dedicated pass

Every earlier phase made one deliberate security decision in context (Phase 1's `.gitignore`,
Phase 3's token trade-off, Phase 11's tenant scoping). This phase closes the gaps that only become
visible once the *whole* app exists — and revisits the trade-offs earlier phases explicitly
deferred.

---

## 3. Two-Factor Authentication

```powershell
composer require laravel/fortify
php artisan vendor:publish --provider="Laravel\Fortify\FortifyServiceProvider"
```
Enable `Features::twoFactorAuthentication()` in `config/fortify.php`. Add a settings screen where
a user scans a QR code (Fortify generates the TOTP secret) with an authenticator app, then must
enter a 6-digit code on future logins. Make it **mandatory for the `admin` role** — the accounts
that can void journal entries or approve Phase 19's high-value agent actions are exactly the ones
where a stolen password shouldn't be enough.

---

## 4. Encryption at rest for sensitive config

Phase 15 stores ETA API credentials and references a signing certificate; Phase 17 stores webhook
secrets. None of these should sit in the database as plain text.

```php
// On the model — Laravel encrypts/decrypts transparently using APP_KEY.
protected function casts(): array
{
    return [
        'webhook_secret' => 'encrypted',
        'eta_api_credentials' => 'encrypted:array',
    ];
}
```
*Why this matters even though the database itself might be access-controlled:* encryption at rest
means a database backup, a leaked `.sql` dump, or a misconfigured read-replica doesn't hand over
usable secrets — it's a second layer, assuming the first (access control) eventually fails, which
is exactly the assumption a security review should make.

**Rotate `APP_KEY` carefully:** it decrypts everything cast as `encrypted`. Losing it means losing
access to that data permanently — back it up as securely as the database itself, never in Git.

---

## 5. Revisit Phase 3's token-storage trade-off

Phase 3 flagged storing the Sanctum token in `localStorage` as a known XSS-exposure trade-off. Now
that the whole app (and its dependencies) exists, actually evaluate it:
- Run `npm audit` — any XSS-capable vulnerability in a dependency is now a real localStorage-token
  theft vector, not a hypothetical one.
- If you want to close this properly: switch to Sanctum's **SPA cookie authentication**
  (httpOnly, same-site cookies instead of a bearer token) — this requires your frontend and
  backend to share a top-level domain in production (e.g. `app.eaccounting.com` and
  `api.eaccounting.com`), a Phase 21 deployment decision. Documenting this dependency here, not
  silently, is the point of a dedicated review phase.

---

## 6. Dependency audits — run both sides

```powershell
composer audit
npm audit
```
Fix anything flagged **critical** or **high** immediately. For **moderate/low**, read what the
vulnerability actually requires to be exploitable (many require an attack vector this specific app
doesn't expose) before spending time on a package upgrade that might introduce breaking changes.
Add both commands to your CI pipeline (Phase 21) so this isn't a one-time check.

---

## 7. Secure HTTP headers

`app/Http/Middleware/SecurityHeaders.php`, applied globally:
```php
class SecurityHeaders
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);
        $response->headers->set('X-Frame-Options', 'DENY'); // no embedding this app in an iframe elsewhere
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
        $response->headers->set(
            'Content-Security-Policy',
            "default-src 'self'; connect-src 'self' " . config('app.frontend_url')
        );
        return $response;
    }
}
```

---

## 8. A quick pass through the classics, against what you actually built

| Risk | Where to check | What you should find |
|---|---|---|
| SQL injection | Every controller/service | Should be Eloquent/query builder throughout — no raw string-interpolated SQL. Grep for `DB::raw(` and `->whereRaw(` and read each one carefully. |
| Mass assignment | Every model's `$fillable` | Should never include `id`, `tenant_id`, `status` (on JournalEntry — that's `PostingService`'s job alone), or `role`. |
| XSS | Vue templates | Vue escapes `{{ }}` by default. Grep for `v-html` — every use is a place raw content could execute as script; confirm each is genuinely necessary and the content isn't user-supplied. |
| CSRF | `routes/web.php` (if any remain) | Stateless API routes with Bearer tokens (Phase 3) aren't CSRF-vulnerable the way cookie-session routes are — but any Blade/web route you added later needs Laravel's default CSRF protection intact. |
| IDOR (viewing someone else's record by guessing an ID) | Every `show`/`update`/`destroy` route | Phase 11's `TenantScope` should make this structurally impossible for cross-tenant access — verify with a real test: log in as Tenant A, try `GET /invoices/{an id belonging to Tenant B}`, confirm a 404, not the record. |
| Insecure agent actions | Phase 19's action classes | Re-read `requiresApproval()` on every single one — is the threshold still right now that real money moves through this system? |

---

## 9. What you built, and why it matters later

- A **repeatable checklist**, not a one-time fix — run Sections 6 and 8's checks again before
  every future major release, not just once now.
- Explicit documentation of the Phase 3 token trade-off's real cost/fix, instead of it quietly
  remaining a known-but-forgotten issue.

## 10. Next: Phase 21

The last phase — taking this from "runs on my machine" to a real, deployed, HTTPS-secured cloud
application.
