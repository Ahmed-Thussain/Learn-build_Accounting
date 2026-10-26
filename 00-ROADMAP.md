# E-Accounting Platform — Learning Roadmap & Architecture

> **What this file is:** the map for the whole project. Read this once before touching code.
> Every other file in this series is one phase from this map, expanded into full steps + code.
> **How we'll teach the "both" answers you gave:** you asked to learn *both* single-tenant and
> multi-tenant SaaS, and *both* a simple installable PWA shell and a full offline-first data layer.
> Rather than build four parallel versions, we build the **simple, real version first**, ship it,
> then **evolve it** into the advanced version in a dedicated later phase. That's not a teaching
> trick — it's genuinely how you'd do this as a professional: you don't design for multi-tenancy
> and offline sync on day one, you design *so that adding them later doesn't require a rewrite*,
> then you add them. You'll see both the "before" and "after" code for each.

---

## 1. What we're building (recap, so we build the right thing)

An accounting system that:
- Runs as a Laravel API backend + Vue 3 SPA frontend, installable as a **PWA** on desktop.
- Starts **single-tenant** (like abicx-erp today), then gets a **multi-tenant SaaS** layer added.
- Starts as an **installable shell** (mostly online), then gets **full offline entry + sync**.
- Covers the core accounting modules: Chart of Accounts, double-entry Journal, General Ledger,
  Accounts Receivable/Payable, Invoicing, Multi-currency, Bank & Cheque management, Financial
  Reports (Trial Balance, P&L, Balance Sheet, Cash Flow).
- Connects to **Egypt's ETA** (Egyptian Tax Authority) for **e-Invoice (B2B)** and **e-Receipt
  (B2C)**, built behind an interface so KSA/UAE/other MENA tax authorities can be added later
  without touching core code.
- Prints **cheques** formatted precisely for Egyptian bank cheque leaves.
- Supports **multi-currency** with a daily exchange-rate job.
- Exposes an **extension point** for e-commerce platforms (WooCommerce/Shopify) to push
  orders/customers in.
- Has an **AI assistant** and **autonomous "autowork" agents** that can act inside the app —
  built with hard permission scopes, a full audit log, and approval gates on anything financial
  or destructive, so autonomy never means "no oversight." (More on why this matters in Phase 20.)
- Ships with **cron jobs / queues** for everything that shouldn't block a user's click (FX rates,
  recurring invoices, tax submissions, report generation).
- Is translated into multiple languages, with real **Arabic RTL** support (not just translated
  strings — actual right-to-left layout).

---

## 2. Tech stack — decisions made now, and why

I picked concrete tools instead of leaving them open, because "TBD" tech choices are the #1 way
a learning project stalls. If you want to swap something later, everything below is chosen
specifically so swapping is possible without a rewrite.

| Layer | Choice | Why |
|---|---|---|
| Backend framework | **Laravel 13** (PHP 8.3+) | Current stable major as of 2026; you already read Laravel code daily in abicx-erp, so patterns will feel familiar fast. |
| Frontend framework | **Vue 3** (Composition API) + **Vite** | Vite is Laravel's own default frontend tooling now — one less thing to configure. Composition API is the modern, more testable way to write Vue and is what you'll want for the offline/sync logic later. |
| Backend ↔ Frontend | **Decoupled REST API + SPA** (not Inertia, not Blade-only) | This is the single most important architecture decision. A PWA that needs to work offline, cache data locally, and sync later **must** talk to the backend as a client, over a well-defined API — it can't depend on the server rendering HTML for every click. Inertia is excellent, but it blurs that boundary in ways that fight you later when you build the offline layer. |
| API auth | **Laravel Sanctum** (token-based) | Built for exactly this: SPA + mobile-style token auth, first-party, no OAuth ceremony needed for now. |
| Database | **MySQL / MariaDB** | Matches your existing ERP stack, and Laragon (see Phase 1) gives you it for free with almost no setup. |
| State management | **Pinia** | Vue's official state library; also where the offline "pending sync" queue will live later. |
| Routing (frontend) | **Vue Router** | Standard pairing with Vue 3 SPA. |
| Permissions/roles | **spatie/laravel-permission** | The de-facto standard package; saves you writing RBAC from scratch so you can focus on accounting logic. |
| Audit logging | **spatie/laravel-activitylog** | Every model change gets a "who did what, when" trail — mandatory for accounting software, and the backbone of the AI-agent audit trail in Phase 20. |
| PDF generation (reports, cheques) | **mPDF** or **TCPDF** | Chosen over dompdf specifically because cheque printing needs *exact millimeter-level coordinate placement* on a pre-printed cheque leaf — mPDF/TCPDF give you that control; dompdf is HTML/CSS-driven and fights you on precision. |
| Offline local storage | **IndexedDB via Dexie.js** | Introduced in Phase 14. Dexie makes IndexedDB usable without hand-rolling low-level browser APIs. |
| i18n | **Laravel lang files** (backend) + **vue-i18n** (frontend) | Standard pairing; both support RTL-aware formatting. |
| Queues/scheduling | **Laravel Queues** (database or Redis driver) + **Laravel Scheduler** | Native to Laravel, no extra infrastructure required to start (database driver first, Redis later if you need speed). |

**Assumption stated, flag it if wrong:** I'm assuming you want the frontend and backend as two
projects in one Git repo (a "monorepo") rather than two separate repos, because while learning,
being able to see backend and frontend side-by-side in one VS Code window is worth more than the
purity of separate repos. This is easy to split later if you ever need independent deploy
pipelines.

---

## 3. Why this build order (the logic, not just the list)

Three rules drove the ordering:

1. **You can't secure, report on, or sync data structures that don't exist yet.** So: schema →
   auth → business logic → API → UI → advanced features, every time.
2. **Every "advanced" version (multi-tenant, offline-first) is taught as an evolution of a working
   simple version**, never built in parallel. You always have something that runs at the end of
   each phase.
3. **Anything that touches real money or an external authority (tax submission, cheque printing,
   AI actions) comes after the core ledger is solid and audited**, never before — mistakes there
   are expensive, so the foundation has to be trustworthy first.

---

## 4. The phases

Each phase becomes its own `.md` file, numbered to match. `[B]`=backend, `[F]`=frontend,
`[A]`=API, `[S]`=security-focused content included in that phase.

| # | Phase | Tags | What you'll have running at the end |
|---|---|---|---|
| 1 | Environment & Project Bootstrap | B F | Laravel API + Vue SPA, both running locally, talking to each other via one real endpoint. |
| 2 | Database Design — Core Accounting Schema (single-tenant) | B | Migrations for Chart of Accounts, Journal Entries, Ledger, Contacts — the spine everything else sits on. |
| 3 | Authentication & Authorization | B A S | Sanctum login/logout, roles & permissions, protected routes. |
| 4 | The Double-Entry Accounting Engine | B | A `PostingService` that enforces debits = credits, posts to the ledger, and can't be bypassed. |
| 5 | REST API Design | A S | Versioned, validated, consistently-shaped API (Form Requests, API Resources, error format). |
| 6 | Frontend Foundation | F | Vue app structured with Pinia stores, router guards, an API client layer, auth flow wired up. |
| 7 | Core Accounting UI | F | Chart of Accounts management, journal entry form, ledger viewer — the first *usable* screens. |
| 8 | Invoicing & AR/AP | B F A | Create/send invoices, record payments, track what's owed and owing. |
| 9 | Multi-Currency | B | Currency table, daily FX-rate job, dual-amount storage (transaction + functional currency). |
| 10 | Financial Reports | B F | Trial Balance, P&L, Balance Sheet, Cash Flow — generated from the ledger, not hand-maintained. |
| 11 | Multi-Tenancy — Evolving Single→Multi | B S | The same app now serves many businesses: `tenant_id` scoping, global scopes, tenant-aware auth, and a note on the database-per-tenant alternative. |
| 12 | PWA Basics — Installable Shell | F | Manifest, service worker, "Add to desktop," works with a flaky connection for *reading* data. |
| 13 | Offline-First Data Layer | F B | IndexedDB (Dexie) local store, a sync queue, conflict resolution — full offline *entry*, not just viewing. |
| 14 | Localization & RTL | B F | Arabic/English (extendable) with real RTL layout, not just mirrored text. |
| 15 | Egypt Tax Authority Integration | B A S | e-Invoice (B2B) and e-Receipt (B2C) submission to ETA, built behind a swappable `TaxAuthority` interface. |
| 16 | Cheque Printing | B F | PDF cheque templates calibrated to real Egyptian bank leaf layouts, printable straight onto the cheque. |
| 17 | E-Commerce Extension | B A | Webhook/connector architecture so WooCommerce/Shopify orders create invoices automatically. |
| 18 | Cron Jobs & Queues | B | Scheduled FX refresh, recurring invoices, retry logic for failed tax submissions. |
| 19 | AI Assistant & Autonomous Agents | B A S | Chat-based assistant *and* scoped autonomous agents, with permission boundaries, dry-run mode, and a full audit trail before anything financial is auto-posted. |
| 20 | Security Hardening Pass | B F S | A dedicated pass: encryption at rest for tax certificates, 2FA, rate limiting, OWASP checklist against what you've built. |
| 21 | Deployment | B F | Cloud deployment (API + SPA + PWA), HTTPS, environment separation (staging/production). |
| 22 | MCP Server — Expose the App to AI Clients | B A S | `laravel/mcp` server (tools/resources/prompts) reusing Phase 19's approval pipeline, per-tenant Sanctum auth, redeployed on Phase 21 infra. |

That's 22 phases after this roadmap. Some will be short, some (11, 13, 15, 19) are genuinely
large and may become 2 files each when we get there — I'll split them if so rather than cram.

---

## 5. A note on your machine (Windows 11, 24GB RAM, C: nearly full)

This actually changes a real decision in Phase 1: with only ~6.5GB free on `C:`, **Docker
Desktop is a bad fit right now** — its WSL2 virtual disk alone commonly grows past that. So
instead of Laravel Sail (Docker-based), Phase 1 sets you up with **Laragon** installed on `D:`,
which gives you PHP, MySQL, and a local domain manager natively, with a tiny footprint. If you
free up C: space later or get a bigger drive, moving to Sail/Docker is a documented, optional
switch — not required.

---

## 6. What's next

Phase 1 (`01-environment-and-project-bootstrap.md`) is ready now — it takes you from an empty
machine to a Laravel API and a Vue app both running and talking to each other. After you've
worked through it (actually run the commands, not just read them), tell me and we'll do Phase 2.

I'm building these one or two phases at a time rather than all 21 at once — at this depth
(full code + explanations + architecture reasoning per phase), generating everything up front
would mean shallower content or you waiting a very long time for one giant file. Going phase by
phase also matches how you said you want to learn: by building.
