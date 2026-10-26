# Phase 21 — Deployment

**Tags:** `[Backend]` `[Frontend]`
**Depends on:** everything — this ships the whole app.
**You'll finish this phase with:** the API and SPA running on real infrastructure, over HTTPS,
with staging and production kept separate, queue/scheduler running as real background services,
and the PWA actually installable from the live URL (which requires HTTPS to work at all).

---

## 1. Learning objectives

- Understand why **HTTPS isn't optional** for this app specifically — service workers (Phase 12)
  and Sanctum cookie auth (if you took Phase 20's recommendation) both refuse to work without it.
- Set up a **process manager** so the queue worker and scheduler (Phase 18) survive server
  reboots and crashes, instead of dying the moment your SSH session closes.
- Keep **staging and production** genuinely separate — different databases, different `.env`,
  different tax-authority environment (sandbox vs. real ETA).

## 2. Choosing where this runs

For a learning project evolving toward a real product, a small VPS (DigitalOcean, Hetzner, or
similar) gives you the most transferable, from-scratch understanding of every piece — you're
setting up the exact same Nginx/PHP-FPM/MySQL/queue-worker stack a "real" production Laravel app
uses, just smaller. (Managed platforms like Laravel Forge or Vapor automate much of what follows —
genuinely worth using once you understand what they're automating, which is the point of doing it
by hand once first.)

---

## 3. Server-side layout

```
/var/www/eaccounting/
├── backend/        ← Laravel, served by PHP-FPM behind Nginx
└── frontend-dist/  ← the built Vue app (static files), served by Nginx directly
```

Build the frontend for production **before** deploying it — this is different from Phase 1's
`npm run dev`:
```powershell
cd frontend
npm run build   # outputs to frontend/dist/ — a folder of plain HTML/CSS/JS, no Node server needed to run it
```

`frontend/.env.production`:
```env
VITE_API_BASE_URL=https://api.eaccounting.com/api/v1
```

---

## 4. Nginx: two server blocks

```nginx
# api.eaccounting.com — Laravel
server {
    listen 443 ssl;
    server_name api.eaccounting.com;
    root /var/www/eaccounting/backend/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

# app.eaccounting.com — the built Vue SPA
server {
    listen 443 ssl;
    server_name app.eaccounting.com;
    root /var/www/eaccounting/frontend-dist;

    location / {
        try_files $uri $uri/ /index.html; # SPA fallback — Vue Router handles the actual routing
    }
}
```
*Why two subdomains on the same top-level domain (`eaccounting.com`):* this is exactly what
Phase 20's optional Sanctum-cookie-auth upgrade needs to work — cookies can be shared across
subdomains of the same registrable domain, but not across genuinely different domains.

---

## 5. HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.eaccounting.com -d app.eaccounting.com
```
Certbot configures auto-renewal — confirm it with `sudo certbot renew --dry-run`. This single step
is what makes Phase 12's PWA installable in the real world (service workers require a secure
context — `https://` or `localhost` only, by browser design, no exceptions).

---

## 6. Keeping the queue worker and scheduler alive: Supervisor

```ini
# /etc/supervisor/conf.d/eaccounting-worker.conf
[program:eaccounting-worker]
command=php /var/www/eaccounting/backend/artisan queue:work --tries=3 --sleep=3
autostart=true
autorestart=true
numprocs=2
user=www-data
```
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start eaccounting-worker:*
```
For the scheduler, add a real cron entry (Linux has native cron, unlike Phase 18's Windows
workaround):
```
* * * * * cd /var/www/eaccounting/backend && php artisan schedule:run >> /dev/null 2>&1
```

---

## 7. Environment separation

Never point staging at production data, and never let staging accidentally submit real documents
to Egypt's tax authority:
```env
# backend/.env (production)
APP_ENV=production
APP_DEBUG=false
ETA_ENVIRONMENT=production
ETA_BASE_URL=https://api.invoicing.eta.gov.eg

# backend/.env (staging)
APP_ENV=staging
APP_DEBUG=false
ETA_ENVIRONMENT=sandbox
ETA_BASE_URL=<eta's sandbox/pre-production URL — confirm current value from ETA's developer portal>
```
Separate databases, separate `APP_KEY` (never share an encryption key between environments),
separate queue — a staging bug should never be able to touch a real customer's ledger or submit a
real e-Invoice.

---

## 8. A minimal CI pipeline

`.github/workflows/deploy.yml` — the smallest version worth having:
```yaml
name: Test and Deploy
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd backend && composer install && php artisan test
      - run: cd frontend && npm install && npm run build
      - run: cd backend && composer audit          # Phase 20's check, now automatic
      - run: cd frontend && npm audit --audit-level=high
  # A deploy step (SSH + rsync, or a platform-specific action) goes here once
  # tests pass — deliberately left as your next step once this pipeline is green.
```

---

## 9. What you built, and why it matters

You've now taken this from a `php artisan serve` window on your own PC (Phase 1) to a deployed,
HTTPS-secured, background-job-capable, environment-separated real application — every phase in
between (the ledger, tenancy, tax integration, offline PWA, AI agents, security hardening) is
running for real, not just on `localhost`.

---

## 10. Next: Phase 22

Deployment isn't the end anymore — Phase 22 adds an MCP server (`laravel/mcp`) on top of what
you just deployed, exposing tools/resources/prompts to external AI clients through the same
auth, tenancy, and approval pipeline, with no new infrastructure.
