# ROADMAP — EAccounting Learning Path

Step-by-step guides for building **EAccounting**, an accounting system for Egyptian SMEs: a
Laravel 13 API with a Vue 3 PWA frontend. Each file is one phase, with full steps, code, and the
reasons behind each decision. The app built from these guides lives in
[`../EAccounting/`](../EAccounting/README.md).

## How to use it

1. Read [00-ROADMAP.md](00-ROADMAP.md) first. It covers what we're building, the tech stack and
   why each piece was chosen, and why the phases come in this order.
2. Work through the phases in order. Each one starts with **Tags**, **Depends on** and
   **You'll finish this phase with**, so you can check you're ready before starting.
3. Run the commands as you go. Every phase ends with something working that the next phase
   builds on.

The simple version of each feature comes first and gets extended later: single-tenant becomes
multi-tenant in Phase 11, and the installable PWA becomes fully offline in Phase 13.

## Phases

| # | Phase | Area |
|---|---|---|
| 00 | [Roadmap & architecture](00-ROADMAP.md) | Overview |
| 01 | [Environment & project bootstrap](01-environment-and-project-bootstrap.md) | Backend, Frontend |
| 02 | [Database schema: core accounting](02-database-schema-core-accounting.md) | Backend |
| 03 | [Authentication & authorization](03-authentication-and-authorization.md) | Backend, API, Security |
| 04 | [Double-entry accounting engine](04-double-entry-accounting-engine.md) | Backend |
| 05 | [REST API design](05-rest-api-design.md) | API, Security |
| 06 | [Frontend foundation](06-frontend-foundation.md) | Frontend |
| 07 | [Core accounting UI](07-core-accounting-ui.md) | Frontend, API |
| 08 | [Invoicing & AR/AP](08-invoicing-and-ar-ap.md) | Backend, Frontend, API |
| 09 | [Multi-currency](09-multi-currency.md) | Backend |
| 10 | [Financial reports](10-financial-reports.md) | Backend, Frontend |
| 11 | [Multi-tenancy (single → multi)](11-multi-tenancy.md) | Backend, Security |
| 12 | [PWA: installable shell](12-pwa-installable-shell.md) | Frontend |
| 13 | [Offline-first data layer](13-offline-first-data-layer.md) | Frontend, Backend |
| 14 | [Localization & RTL](14-localization-and-rtl.md) | Backend, Frontend |
| 15 | [Egypt Tax Authority (ETA) integration](15-egypt-tax-authority-integration.md) | Backend, API, Security |
| 16 | [Cheque printing](16-cheque-printing.md) | Backend, Frontend |
| 17 | [E-commerce extension](17-ecommerce-extension.md) | Backend, API |
| 18 | [Cron jobs & queues](18-cron-jobs-and-queues.md) | Backend |
| 19 | [AI assistant & autonomous agents](19-ai-assistant-and-autonomous-agents.md) | Backend, API, Security |
| 20 | [Security hardening pass](20-security-hardening-pass.md) | Backend, Frontend, Security |
| 21 | [Deployment](21-deployment.md) | Backend, Frontend |
| 22 | [MCP server](22-mcp-server.md) | Backend, API, Security |

## Phase groups

- **Core (1–10):** setup, schema, auth, the posting engine, the API, the UI, invoicing,
  multi-currency, and reports. After Phase 10 you have a working single-tenant accounting app.
- **Upgrades (11–14):** multi-tenancy, PWA, offline sync, Arabic/RTL.
- **Integrations (15–19):** ETA e-Invoice/e-Receipt, cheque printing, WooCommerce/Shopify,
  background jobs, and the AI assistant.
- **Shipping (20–22):** security review, deployment, and the MCP server for external AI clients.

## Build progress

See the **Roadmap status** table in [`../EAccounting/README.md`](../EAccounting/README.md#roadmap-status).
Phase 23 (social inbox) was built in the app but has no guide in this folder yet.

## Other files

- `eaccounting-learning-path.zip` is an older copy of phases 00–21. It doesn't include Phase 22.
  Use the `.md` files in this folder.
