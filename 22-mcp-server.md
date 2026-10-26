# Phase 22 — MCP Server (Expose the App to AI Clients)

**Tags:** `[Backend]` `[API]` `[Security]`
**Depends on:** everything through Phase 21 — this phase deliberately comes *after* deployment,
because it exposes what you already built (ledger, API, AI-agent actions) to *external* AI
clients over an open standard, and that is only safe once auth, tenancy, audit, and HTTPS all
exist for real.
**You'll finish this phase with:** a Model Context Protocol server built with Laravel's official
`laravel/mcp` package — read-only tools and resources first, then guarded write tools that reuse
Phase 19's approval pipeline — connectable from Claude, Cursor, or any MCP-compatible client,
scoped per tenant, fully audited, and redeployed on your Phase 21 infrastructure with no rewrite.

---

## 1. Learning objectives

By the end of this phase you should be able to explain, in your own words:

- What **MCP (Model Context Protocol)** is and why it replaces one-off AI integrations: one
  standard server, any compatible client (Claude Desktop, Cursor, Copilot, ChatGPT) can connect —
  no custom plugin per assistant.
- The three things an MCP server exposes — **Tools** (things the agent can *call*), **Resources**
  (data the agent can *read* by URI), **Prompts** (reusable conversation templates) — and which of
  your accounting capabilities belongs in each.
- Why your MCP write tools must **never bypass Phase 19's `AgentActionDispatcher`** — the MCP
  layer is a new *door* into the app, not a new *code path* around its safety checks.
- How **tenant scoping + token auth + HTTPS** (Phases 11, 3, 21) combine to make a remotely
  reachable AI entry point safe instead of terrifying.

## 2. Why this phase comes after deployment

Phase 19 built an AI assistant that lives *inside* your app — your frontend calls your LLM
integration, which calls your allow-listed actions. MCP flips that around: an *outside* agent
(Claude running on someone's laptop, an IDE agent, another company's automation) connects *into*
your app and asks to read data or request actions.

That is a strictly more exposed posture, so it belongs after:

- **Phase 11 (tenancy)** — every MCP request must resolve to exactly one tenant, or Tenant A can
  read Tenant B's ledger through a clever prompt.
- **Phase 19 (agent actions + approval queue)** — you already have the allow-list, dry-run, and
  approval semantics; MCP reuses them instead of inventing weaker ones.
- **Phase 20 (security pass)** — token handling, encryption at rest, and the audit checklist were
  practiced there; MCP tokens get the same treatment.
- **Phase 21 (deployment + HTTPS)** — remote MCP clients refuse non-HTTPS endpoints in practice,
  and the `web` transport is just another HTTPS route on your existing Nginx setup. Nothing new
  to provision — you extend what you deployed, then redeploy.

*Principle, same as Phases 11 and 13:* this is an **evolution of a working system**, not a
parallel build. At the end of every section below, the app still runs exactly as before — with
one new route (`/mcp/...`) added.

---

## 3. Section A — Install the official package and register your first server

Laravel ships an official MCP package (current as of 2026 — if the version below has moved on,
use the latest `13.*`-compatible release and read its changelog the same way Phase 20 taught
you to read `composer audit` output):

```powershell
composer require laravel/mcp
php artisan vendor:publish --provider="Laravel\Mcp\McpServiceProvider"
```

This publishes `routes/ai.php` — the MCP equivalent of `routes/api.php` from Phase 1. It exists
separately on purpose: MCP routes have different auth, throttling, and versioning needs than
your REST API, so they get their own file instead of being mixed in.

```powershell
php artisan make:mcp-server AccountingServer
```

This creates `app/Mcp/Servers/AccountingServer.php`:

```php
<?php

namespace App\Mcp\Servers;

use Laravel\Mcp\Server;
use Laravel\Mcp\Attributes\Version;
use Laravel\Mcp\Attributes\Description;

#[Version('1.0.0')]
#[Description('E-Accounting: chart of accounts, invoices, and reports per tenant.')]
class AccountingServer extends Server
{
    // Tools, resources, and prompts register here (Sections 4–6).
    // Start EMPTY — Section 4 adds the first read-only tool,
    // and every write tool in Section 5 goes through Phase 19's dispatcher.
    protected array $tools = [
        //
    ];

    protected array $resources = [
        //
    ];

    protected array $prompts = [
        //
    ];
}
```

Register it in `routes/ai.php` with **both** transports, and understand what each is for:

```php
<?php

use App\Mcp\Servers\AccountingServer;
use Laravel\Mcp\Facades\Mcp;

Mcp::web('/mcp/accounting', AccountingServer::class)
    ->middleware(['auth:sanctum', 'throttle:mcp']);

Mcp::local('accounting', AccountingServer::class);
```

- **`web`** — HTTP POST transport. This is what remote clients (a hosted Claude, a teammate's
  Cursor) connect to. It runs through your existing Phase 21 Nginx + HTTPS setup — no new
  server process, no Supervisor entry (unlike Phase 18's queue worker). Stateless, like the
  rest of `routes/api.php`.
- **`local`** — stdio transport, run as an Artisan command. This is for *your own machine only*:
  Claude Desktop or your IDE spawns `php artisan mcp:start accounting` as a subprocess and talks
  to it over stdin/stdout. Nothing leaves the laptop; the fastest way to develop and test.

*Why both from day one:* you develop against `local` (no auth ceremony, no network, instant
feedback with the MCP Inspector below), and you expose `web` only once Sections 7's auth and
tenant scoping are proven. Same instinct as Phase 1's "prove it locally before deploying it."

---

## 4. Section B — Tools: start read-only, reuse what the REST API already proves

A Tool is a callable function: it declares an input **schema** and implements **handle()**.
Build the read-only tools first — they can't move money, so they're the safe place to learn the
pattern, and they're immediately useful ("ask Claude what's overdue").

```powershell
php artisan make:mcp-tool ListOverdueInvoicesTool
```

```php
<?php

namespace App\Mcp\Tools;

use Laravel\Mcp\Tool;
use Laravel\Mcp\Attributes\Description;

#[Description('List overdue sales invoices for the current tenant, newest-due first.')]
class ListOverdueInvoicesTool extends Tool
{
    public function schema(): array
    {
        // Same discipline as Phase 5's Form Requests: the schema IS the validation.
        // The client cannot send anything this doesn't describe.
        return [
            'type' => 'object',
            'properties' => [
                'limit' => ['type' => 'integer', 'minimum' => 1, 'maximum' => 100, 'default' => 20],
            ],
        ];
    }

    public function handle(array $arguments): string
    {
        // Phase 11's global TenantScope applies here automatically — there is no
        // code path in this method that can reach another tenant's rows, because
        // the scope is on the model, not on the caller remembering to filter.
        // (Verify it the same way Phase 20's IDOR check does: Tenant A token,
        // Tenant B invoice id, confirm it never appears.)
        $invoices = \App\Models\Invoice::query()
            ->where('status', 'overdue')
            ->orderBy('due_date', 'desc')
            ->limit($arguments['limit'] ?? 20)
            ->get(['id', 'number', 'contact_id', 'total', 'currency_code', 'due_date']);

        return $invoices->toJson();
    }
}
```

Register it:

```php
protected array $tools = [
    \App\Mcp\Tools\ListOverdueInvoicesTool::class,
];
```

Suggested first batch (all read-only, all thin wrappers over queries your API already makes):

| Tool | What it returns | Reuses |
|---|---|---|
| `ListOverdueInvoicesTool` | overdue invoices + contact + total | Phase 8's invoice query |
| `TrialBalanceTool` | trial balance as of a date | Phase 10's report service |
| `AccountBalanceTool` | balance of one account code | Phase 4's ledger |
| `FindContactTool` | contacts matching a name | Phase 8's contacts |

*Rule that carries the whole phase:* a Tool never contains business logic. It validates input,
calls the same service/query the REST controller calls, and serializes the result. If you find
yourself writing posting logic inside `handle()`, stop — that logic belongs in the service, and
the tool calls the service. (This is the Phase 5 "controllers stay thin" lesson, applied to a
new kind of caller.)

---

## 5. Section C — Write tools: the MCP door into Phase 19's pipeline, not around it

Read tools are safe by construction. Write tools are where money moves — so they don't get their
own logic at all. Each one is a ~20-line adapter from "MCP arguments in" to
"`AgentActionDispatcher::request()` and report the status back":

```php
<?php

namespace App\Mcp\Tools;

use App\Services\AgentActionDispatcher;
use Laravel\Mcp\Tool;
use Laravel\Mcp\Attributes\Description;

#[Description('Request creation of a draft sales invoice. May require human approval; check action_status.')]
class CreateDraftInvoiceTool extends Tool
{
    public function schema(): array
    {
        return [
            'type' => 'object',
            'required' => ['contact_id', 'lines'],
            'properties' => [
                'contact_id' => ['type' => 'integer'],
                'lines' => [
                    'type' => 'array',
                    'items' => [
                        'type' => 'object',
                        'required' => ['description', 'quantity', 'unit_price'],
                        'properties' => [
                            'description' => ['type' => 'string', 'maxLength' => 255],
                            'quantity' => ['type' => 'number', 'exclusiveMinimum' => 0],
                            'unit_price' => ['type' => 'number', 'exclusiveMinimum' => 0],
                        ],
                    ],
                ],
            ],
        ];
    }

    public function handle(array $arguments): string
    {
        // THE load-bearing line of this phase: the MCP tool does not create
        // anything itself. It asks Phase 19's dispatcher, which runs preview(),
        // requiresApproval(), execution, and activity() logging — the exact same
        // path as the in-app chat assistant and the autonomous scheduler.
        // There is one pipeline for all agent-triggered writes, regardless of
        // which client (in-app chat vs. external MCP client) asked.
        $request = app(AgentActionDispatcher::class)->request(
            'create_draft_invoice',
            $arguments,
            requestedByUserId: $this->context->user()->id, // the Sanctum token owner, Section 7
        );

        return json_encode([
            'action_status' => $request->status, // "auto_executed" | "pending_approval" | "executed"
            'preview' => $request->preview_result, // dry-run: what WOULD happen (Phase 19 §3)
            'approval_id' => $request->status === 'pending_approval' ? $request->id : null,
            // The frontend screen from Phase 19 §7 already lists this id —
            // no new approval UI needed. The MCP client tells the human
            // "this needs your approval in the app," and the app shows it.
        ]);
    }
}
```

Consequences worth stating explicitly, because they answer "without compromising the app":

1. **No new permission model.** The tool runs as the Sanctum token owner (`$this->context->user()`);
   Phase 3's policies and `requiresApproval()` thresholds apply unchanged. An MCP client can never
   do something the token owner couldn't do in the UI.
2. **Destructive/financial actions gate the same way.** `post_journal_entry` above a threshold,
   anything touching a sensitive account, any tax submission — `pending_approval` in the same
   `agent_action_requests` table, approved in the same Vue screen.
3. **Audit is free.** `runAndLog()` already writes the `activity('agent')` entry with action name,
   params, and result — add the MCP client name to `withProperties()` so you can later query "which
   actions came via MCP vs. in-app chat."

Start with **at most two** write tools (`CreateDraftInvoiceTool`, `CreateJournalEntryTool` capped
to drafts), and only widen the allow-list once the approval queue has been exercised for real.

---

## 6. Section D — Resources and Prompts: context and repeatable workflows

Tools *do* things. Resources give the agent *context to reason with*, addressable by URI.
Prompts package *repeatable workflows* so every client behaves consistently.

**Resources** — expose the stable reference data agents constantly need, read-only:

```php
// app/Mcp/Resources/ChartOfAccountsResource.php (conceptual — see laravel/mcp docs
// for the exact resource class API in your installed version)
//
// URI:  accounting://chart-of-accounts
// Returns: the current tenant's active accounts (code, name, type), cached 60s.
// Why a resource instead of a tool: the agent needs this as background context
// ("which code is Office Rent?") before it ever calls a tool — it reads it
// unprompted, the way a human glances at the CoA screen first (Phase 7).
//
// Others worth adding: accounting://contacts/active,
// accounting://reports/trial-balance/{date}, accounting://tax/eta-status/{invoiceId}
```

**Prompts** — one template per workflow Phase 19's autonomous agents already run, so the
external client and the internal scheduler follow the same steps:

- `month-end-close` — "reconcile these accounts, list unposted drafts, summarize what needs
  approval, never post directly."
- `overdue-invoice-review` — "list overdue invoices, draft reminder text per contact, request
  send-reminder actions (which will gate on approval)."
- `expense-categorization` — "given these uncategorized lines and this CoA, propose account codes
  with reasons; ask before posting."

*Why prompts live server-side:* if the workflow lives in each client's system prompt, five
clients drift into five behaviors. If it lives on your server, every client runs the version
you reviewed — the same reason Phase 5 centralized validation in Form Requests instead of
trusting each frontend form.

---

## 7. Section E — Auth, tenancy, and throttling: the three gates on `routes/ai.php`

An MCP endpoint without these is a public ledger reader. All three are one-line-each because
the hard work happened in earlier phases — this section is *wiring*, not invention:

```php
Mcp::web('/mcp/accounting', AccountingServer::class)
    ->middleware(['auth:sanctum', 'throttle:mcp']);
```

1. **Auth — start with Sanctum tokens (Phase 3), plan for OAuth 2.1 via Passport for
   third parties.** Sanctum bearer tokens are exactly what your Vue app already uses, so personal
   clients (your Claude Desktop, your Cursor) connect today: generate a token in the app's
   settings screen, paste it into the client's MCP config. For *other companies'* agents
   connecting to you, the `laravel/mcp` docs describe `Mcp::oauthRoutes()` + Passport — adopt
   that only when you have a real third-party consumer, the same "evolve when needed" stance as
   Phases 11 and 13. Either way: tokens carry `abilities` (e.g. `mcp:read`, `mcp:request-writes`);
   write tools check the ability before touching the dispatcher.
2. **Tenancy — resolve tenant from the token owner, never from a client-supplied parameter.**
   `currentTenantId()` comes from `$this->context->user()->tenant_id`, the same source Phase
   11's auth uses. If a tool schema contains anything resembling `tenant_id` as an input field,
   delete it — that is the IDOR Phase 20's table warns about, reintroduced through a new door.
3. **Throttling — stricter than the REST API.** Phase 19 already created a `throttle:agent`
   limiter because LLM clients burst. Reuse it for MCP (`throttle:mcp` as an alias or the same
   limiter): an MCP client can fan out dozens of tool calls per user message, so the limit that
   feels generous for humans clicking is far too loose here. Log every tool call (name, tenant,
   user, ms) to the activity log — it doubles as your usage/cost telemetry.

Add one small frontend screen to make this manageable (the only Vue work in this phase):
`frontend/src/views/McpTokensView.vue` — list the user's MCP tokens (name, abilities,
last-used), with Create and Revoke buttons. Same list-and-act pattern as every screen since
Phase 7; the backend is Sanctum's existing `createToken`/`delete` endpoints, scoped to
`mcp:*` abilities.

---

## 8. Section F — Test with the Inspector, then connect a real client

Never debug MCP against a real AI client first — the failure could be your server, the client's
prompting, or the network, and you can't tell which. Isolate the server:

1. **Inspector (server correctness):** run the local transport and point the MCP Inspector at it
   (see the `laravel/mcp` docs for the exact `npx @modelcontextprotocol/inspector` invocation in
   your version). Call each tool with good input, bad input, and cross-tenant input: good input
   returns data, bad input returns schema errors, cross-tenant input returns nothing. Call a write
   tool and confirm the row in `agent_action_requests` is `pending_approval` (not executed) with
   a correct `preview_result` — then approve it in the Phase 19 screen and confirm execution +
   audit entry.
2. **Local client (end-to-end, private):** add the `local` server to Claude Desktop / Cursor
   config (`php artisan mcp:start accounting` as the command), ask it "what's overdue?", confirm
   it calls the tool rather than inventing numbers.
3. **Remote client (the real thing):** deploy per Section 9, add the `https://api.…/mcp/accounting`
   server with the Sanctum token, repeat the same checks over HTTPS — plus revoke the token in
   `McpTokensView.vue` and confirm the client immediately loses access.

---

## 9. Section G — Redeploy: the Phase 21 setup already covers this

No new infrastructure — but a real redeploy, not "it works on localhost":

- **Nginx:** the `web` transport is a normal Laravel route, served by the existing
  `api.eaccounting.com` block from Phase 21. Confirm the SPA-fallback block (`app.…`) doesn't
  swallow `/mcp/*` — API and MCP hostnames keep them naturally separate, which is part of why
  Phase 21 split the subdomains.
- **HTTPS:** required, not recommended — remote MCP clients and the Phase 12 service-worker
  rule share the same "secure context" expectation. Certbot from Phase 21 already covers it.
- **Env separation:** staging gets its own MCP route pointing at the sandbox ETA (Phase 21 §7) —
  an external agent hammering staging must never be able to submit a real e-Invoice. Same
  `.env` split, one more consumer of it.
- **CI:** extend the Phase 21 workflow with the MCP test helpers the package provides (fake tool
  calls, assert resources/prompts — see package docs) so `composer audit` and the tool tests run
  on every push.

```yaml
# addition to .github/workflows/deploy.yml (Phase 21 §8)
- run: cd backend && php artisan test --filter=Mcp
```

---

## 10. What you built, and why it matters

- **One standard door for every AI client** — Tools, Resources, and Prompts via the official
  `laravel/mcp` package, reachable over stdio locally and HTTPS remotely, instead of a bespoke
  integration per assistant.
- **No second permission system.** Reads inherit Phase 11's tenant scoping; writes funnel through
  Phase 19's dispatcher — same allow-list, same approval queue, same audit log, with the client
  name attached. The safety argument from Phase 19 ("one pipeline, no weaker path") now covers
  external clients too.
- **Token discipline the app can actually operate** — Sanctum abilities today, a documented OAuth
  upgrade path for third parties later, a revoke button in the UI, and throttle + activity logging
  that make MCP usage visible instead of mysterious.

## 11. Next: Phase 23

Phase 22 made the app *accessible* to outside agents. The natural next steps from here are the
ones this roadmap has been deferring deliberately: a Fixed Assets module, payroll, and a second
country's tax authority behind Phase 15's interface — all consumers of the ledger, tenancy, and
(now) MCP surface you already have. Pick the one your learning needs next; each is an extension
of a working, deployed, AI-accessible system — not a new foundation.

(End of file - Phase 22)
