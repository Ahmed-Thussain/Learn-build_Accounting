# Phase 19 — AI Assistant & Autonomous Agents

**Tags:** `[Backend]` `[API]` `[Security]`
**Depends on:** everything through Phase 18 — this phase deliberately comes *after* the ledger,
API, and audit-friendly design are solid, not before.
**You'll finish this phase with:** a chat-based assistant that can answer questions and take
real actions in the app, plus autonomous scheduled agents — both built with hard permission
scopes, a full audit trail, and an approval gate on anything financial or irreversible.

---

## 1. Learning objectives

- Understand why "AI that can act in the app" needs the **same authorization model as a human
  user**, not a separate, more-trusted one.
- Build a **dry-run mode**: every agent action can be previewed before it's committed.
- Build an **approval queue** for actions that shouldn't happen without a human confirming them.
- Understand why every agent action must be **fully logged** — not as an afterthought, but as the
  feature that makes autonomy trustworthy at all.

## 2. Why this phase comes last (of the feature phases)

You asked for an AI/"autowork" system that can finish or create things it has access to "without
compromising the app." The honest way to deliver that isn't clever prompting — it's **narrow,
auditable, revocable permissions**, built on infrastructure this roadmap already has: roles
(Phase 3), the immutable ledger (Phase 4), and the audit log package
(spatie/laravel-activitylog, installed now if you haven't already:
`composer require spatie/laravel-activitylog`). An AI agent is, from the system's point of view,
just another actor — it should never be able to do something a human `accountant` role couldn't
also do, and everything it does should be exactly as visible after the fact as a human's actions.

---

## 3. The core design: actions as first-class, permissioned classes

Never let an LLM call your Eloquent models or services directly. Instead, define a fixed,
reviewable **allow-list** of actions it's permitted to request:

```php
interface AgentActionInterface
{
    public function name(): string;                 // e.g. "create_journal_entry"
    public function requiresApproval(array $params): bool;
    public function preview(array $params): array;   // dry-run: what WOULD happen
    public function execute(array $params): array;   // actually do it
}
```

```php
class CreateJournalEntryAction implements AgentActionInterface
{
    public function __construct(private PostingService $postingService) {}

    public function name(): string { return 'create_journal_entry'; }

    public function requiresApproval(array $params): bool
    {
        // A concrete, adjustable policy — not a blanket "AI can never post."
        // Small, routine entries can flow through; anything large or touching
        // a sensitive account always stops for a human.
        $total = collect($params['lines'])->sum('debit');
        return $total > 5000 || $this->touchesSensitiveAccount($params['lines']);
    }

    public function preview(array $params): array
    {
        // Runs the SAME validation Phase 5's Form Request does, without saving anything —
        // "here's what would be created, and whether it would balance" — never a guess.
        $totalDebit = collect($params['lines'])->sum('debit');
        $totalCredit = collect($params['lines'])->sum('credit');
        return [
            'would_balance' => bccomp((string) $totalDebit, (string) $totalCredit, 2) === 0,
            'lines' => $params['lines'],
        ];
    }

    public function execute(array $params): array
    {
        $entry = JournalEntry::create([...]); // same construction as Phase 5's controller
        $entry->lines()->createMany($params['lines']);
        $posted = $this->postingService->post($entry); // Phase 4's guarantee still applies — always
        return ['journal_entry_id' => $posted->id];
    }

    private function touchesSensitiveAccount(array $lines): bool
    {
        $sensitiveCodes = ["3000"]; // Owner's Equity, e.g. — configurable per tenant
        return collect($lines)->contains(fn ($l) => Account::find($l['account_id'])?->code === '3000');
    }
}
```

Register a **fixed** set of these in a service provider — the agent can request `"delete_invoice"`
only if such a class exists and is registered; there is no path from an LLM's text output straight
to your database.

---

## 4. The approval queue

```php
Schema::create('agent_action_requests', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained();
    $table->string('action_name');
    $table->json('params');
    $table->json('preview_result')->nullable();
    $table->enum('status', ['pending_approval', 'approved', 'rejected', 'executed', 'auto_executed']);
    $table->foreignId('requested_by_user_id')->nullable()->constrained('users'); // null = autonomous trigger
    $table->foreignId('approved_by_user_id')->nullable()->constrained('users');
    $table->timestamps();
});
```

```php
class AgentActionDispatcher
{
    public function __construct(private array $actions) {} // injected: name => AgentActionInterface

    public function request(string $actionName, array $params, ?int $requestedByUserId): AgentActionRequest
    {
        $action = $this->actions[$actionName] ?? throw new \RuntimeException("Unknown agent action: {$actionName}");

        $request = AgentActionRequest::create([
            'tenant_id' => currentTenantId(),
            'action_name' => $actionName,
            'params' => $params,
            'preview_result' => $action->preview($params),
            'requested_by_user_id' => $requestedByUserId,
            'status' => $action->requiresApproval($params) ? 'pending_approval' : 'auto_executed',
        ]);

        if ($request->status === 'auto_executed') {
            $this->runAndLog($request, $action);
        }

        return $request;
    }

    public function approve(AgentActionRequest $request, User $approver): void
    {
        $action = $this->actions[$request->action_name];
        $request->update(['status' => 'approved', 'approved_by_user_id' => $approver->id]);
        $this->runAndLog($request, $action);
    }

    private function runAndLog(AgentActionRequest $request, AgentActionInterface $action): void
    {
        $result = $action->execute($request->params);
        $request->update(['status' => 'executed']);

        // This is the whole point: every agent action leaves the exact same
        // kind of audit trail a human's action does, queryable the same way.
        activity('agent')
            ->causedBy($request->requestedByUser ?? 'autonomous-scheduler')
            ->withProperties(['action' => $request->action_name, 'params' => $request->params, 'result' => $result])
            ->log("Agent action executed: {$request->action_name}");
    }
}
```

---

## 5. The chat endpoint

```php
class AssistantController extends Controller
{
    public function chat(Request $request, AgentActionDispatcher $dispatcher)
    {
        $this->authorize('useAssistant', Assistant::class); // a Phase 3-style policy check

        // Send the user's message + a description of ONLY the registered actions
        // (Section 3) to the LLM, using tool/function-calling. The LLM can request
        // an action by name and params — it can never do anything not in that list.
        $llmResponse = $this->llmClient->chat($request->input('message'), tools: $this->availableActionSchemas());

        if ($llmResponse->wantsToCallAction()) {
            $actionRequest = $dispatcher->request(
                $llmResponse->actionName(),
                $llmResponse->actionParams(),
                requestedByUserId: $request->user()->id,
            );

            return response()->json([
                'reply' => $llmResponse->text(),
                'action_status' => $actionRequest->status, // frontend shows "needs your approval" if so
                'preview' => $actionRequest->preview_result,
            ]);
        }

        return response()->json(['reply' => $llmResponse->text()]);
    }
}
```

*Rate limiting matters more here than anywhere else in the app:* apply a stricter throttle
(`throttle:agent`) than the general API limit (Phase 5) — an LLM integration that's bugged or
being probed by a malicious user can generate far more requests, far faster, than a human clicking
through a UI.

---

## 6. Autonomous (scheduled) agents — the same pipeline, no chat involved

An "autowork" job (e.g., "review overdue invoices and draft reminder emails") is just
`Schedule::command('app:run-autonomous-agent --task=overdue-invoice-review')`, which calls
`AgentActionDispatcher::request()` directly instead of via a chat message —
`requestedByUserId: null`, logged as `'autonomous-scheduler'`. **It goes through the exact same
`requiresApproval()` check and the exact same audit log** — there is no separate, less-supervised
code path for autonomous vs. chat-triggered actions. This is the concrete answer to "without
compromising the app": the safety mechanism doesn't depend on remembering to add checks to a new
trigger source, because there's only one path any action can take.

---

## 7. Frontend: reviewing pending actions

`frontend/src/views/AgentApprovalsView.vue` — a list of `pending_approval` requests, each showing
the `preview_result` (Section 3's dry-run output) side-by-side with an Approve/Reject button, and
a link into the resulting `activity` log entry once executed. Same list-and-act pattern as every
earlier screen in this roadmap.

---

## 8. What you built, and why it matters later

- A **closed allow-list of actions**, never free-form model/database access — the actual boundary
  that makes "AI with access to the app" safe, not a prompt telling it to "be careful."
- **One pipeline for both chat-triggered and autonomous actions** — no separate, weaker-checked
  path for scheduled automation.
- A **complete, queryable audit trail**, using the exact package (spatie/laravel-activitylog)
  this roadmap flagged as necessary all the way back at Phase 3.

## 9. Next: Phase 20

With every feature built, Phase 20 is a dedicated security pass across the whole app — the things
worth doing once, deliberately, rather than piecemeal per-phase: 2FA, encryption at rest, a
dependency audit, and a full OWASP-style review.
