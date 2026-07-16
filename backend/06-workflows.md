# 06 — Workflows (Call Automation)

> Backend domain doc. Source: `backend/src/{models/Workflow.model.ts, controllers/workflow.controller.ts, routes/workflow.routes.ts, services/{workflow,workflowRunner,workflowScheduler,workflowFollowupPoller}.service.ts, jobs/{workflowFollowup,workflowScheduler}.{queue,worker}.ts}`.

## 1. Overview

A **Workflow** is a simple linear automation: *when X happens, wait N minutes, then have an agent call this number*. It's the "if this then call" feature — distinct from `Campaign` (bulk-dial a static list) and manual `Call` dispatch. A workflow is a `nodes[]` array acting like a tiny flowchart: always starts with exactly one `trigger` node, followed by any mix of `delay` and `call` nodes.

## 2. Data Model (`models/Workflow.model.ts`)

| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `name` | string | |
| `status` | enum | `active \| inactive` — inactive workflows are simply never matched against triggers |
| `nodes` | `IWorkflowNode[]` | see below |
| `executions` | number | incremented every time the workflow successfully schedules at least one follow-up call |
| `lastExecutedAt` | Date | also used by the scheduler to enforce "at most once per calendar day" for `scheduled`-trigger workflows |

### Node shape
```ts
{ id: string, type: 'trigger'|'delay'|'call', title: string, subtitle: string, config: Map<string,string> }
```
`config` is stored as a Mongoose `Map<String,String>` (**not** a plain object) — every reader in the codebase has to branch on `instanceof Map` vs. plain object (`cfgMap`/`cfgGet`/`extractConfig` helpers duplicated across `workflow.service.ts`, `workflowRunner.service.ts`, `workflowScheduler.service.ts` — worth consolidating into one shared utility during rebuild). Bracket notation (`config['key']`) silently returns `undefined` on a real `Map` instance — a common source of bugs if this detail is missed.

### Node types & their `config` keys
- **`trigger`** (always `nodes[0]`) — `config.triggerType` ∈ `call_completed | call_started | inbound_call | lead_created | scheduled`. The `scheduled` type additionally needs `scheduledPhone`, `scheduledTime` (`HH:MM`), `scheduledTimezone` (IANA, default `Asia/Kolkata`), `scheduledFrequency` (`daily|weekdays|weekly`, default `daily`), `scheduledDays` (comma-separated `mon,fri` for weekly).
- **`delay`** — `config.delayType` ∈ a fixed preset list (`5min,15min,30min,1hour,2hours,3hours,6hours,12hours,24hours,48hours,72hours`), or `custom` (+`delayCustomMinutes`, clamped 1–10080 i.e. max 7 days), or `user_requested` (delay is resolved at trigger-time from an actual callback datetime provided by the trigger context — e.g. "call me back at 3pm" — if no such datetime is available when the trigger fires, the whole workflow is aborted for that firing, not defaulted to anything).
- **`call`** — `config.agentId` (+ denormalized `agentLabel` for display) — which agent places the follow-up call.

## 3. API Reference — `/api/workflows` (JWT)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list; filters `search` (name regex), `status` |
| POST | `/` | Create — `{name, agentId?}`. Always starts with a default `call_completed` trigger node; if `agentId` given, appends one pre-configured `call` node. **Requires the owner to already have a customer-owned (non-platform-pool) phone number** (`assertCustomerOwnedNumberForAutomations` — see `03-calls-realtime-pipeline.md §3` caller-ID policy) — thrown as a validation error at creation time, before the user can even build out the workflow. |
| GET | `/:id` | Detail (403 if not owned) |
| PATCH | `/:id` | Update `name`/`status`/`nodes` — see validation rules below |
| PATCH | `/:id/status` | Convenience endpoint — same as `PATCH /:id` with only `status` |
| DELETE | `/:id` | Delete |

### Node validation & normalization (`workflow.service.ensureNodeShape`)
On any `nodes` write: rejects non-array input, rejects unknown node `type`s, re-derives each node's `title`/`subtitle` display strings from its `config` (so the frontend never has to keep display labels in sync with config changes — the backend is the source of truth for how a node reads), and **hard-requires `nodes[0].type === 'trigger'`** (a workflow can never start with anything else). An empty `nodes` array is coerced to a single default trigger node rather than rejected.

### Activation validation (`workflow.service.validateActivation`, runs when `status` is being set to `'active'`)
1. Re-checks `assertCustomerOwnedNumberForAutomations` (a workflow could have been created, then the customer's only number removed — re-validated at every activation, not just creation).
2. If trigger is `scheduled`: requires both `scheduledPhone` and `scheduledTime` to be set.
3. Requires **at least one** `call` node to exist, and at least one of them to have an `agentId` configured — an activated workflow that would never actually place a call is rejected outright.

## 4. Trigger firing (`workflowRunner.service.ts`)

Two trigger-firing entry points, called from elsewhere in the codebase whenever something trigger-worthy happens:

- **`enqueueWorkflowFollowupsForTrigger({triggerType, userId, to, contactId?, callbackAt?, sourceId?, skipIfWorkflowFollowupCall?})`** — the general-purpose entry point. Called from:
  - `twilio.service.handleStatusCallback` on the first transition to `in-progress` (`call_started`) and via `enqueueWorkflowFollowupsAfterCall` on `completed` (`call_completed`).
  - `twilio.service.handleInboundCall` (`inbound_call`).
  - `lead.service.createLead` (`lead_created`, with `callbackAt: doc.nextFollowUpAt`).
- **`enqueueWorkflowFollowupsAfterCall(callId)`** — thin wrapper used specifically for `call_completed`, resolves the customer's phone from `call.direction` (inbound → `call.from`, outbound → `call.to`) and skips entirely if the call itself was `workflowFollowup:true` (**infinite-loop guard**: a follow-up call completing must never itself schedule another follow-up).

### Firing logic
For `call_completed`/`call_started` triggers, first re-fetches the `Call` and bails if it was itself a `workflowFollowup` call (second layer of the same loop guard, checked here in addition to the caller-side check — belt and suspenders). Then loads every `active` workflow owned by the user, filters to ones whose trigger node's `triggerType` matches, and for each matching workflow walks its node list **in order**:
- `delay` nodes accumulate into a running `cumulativeMinutes` total. If a `user_requested` delay has no resolvable `callbackAt`, the **entire workflow firing is aborted** (not just that node skipped) — logged, no calls scheduled.
- `call` nodes schedule a delayed follow-up job (`scheduleWorkflowFollowupJob`, see §5) using the `cumulativeMinutes` accumulated *so far* — meaning multiple call nodes after different delay nodes in the same workflow fire at different offsets, each based on the delays preceding it.
- If at least one call was scheduled and the workflow wasn't aborted, `Workflow.executions` is incremented and `lastExecutedAt` stamped.

## 5. Scheduling mechanism — Redis sorted set → BullMQ (`workflowFollowupPoller.service.ts`)

This is a **deliberately two-layer design**, not a direct BullMQ delayed job — the code comment explains why: BullMQ v5's delayed-job promotion was found unreliable on Redis Cloud (`volatile-lru` eviction policy issue). Instead:

1. **`scheduleWorkflowFollowupJob(data)`** — computes `fireAt = now + delayMs`, `ZADD`s a JSON payload into a single Redis sorted set (`vx:workflow:followups`) scored by `fireAt`. This is the actual "timer" — durable in Redis, not in-process memory, so it survives an API process restart.
2. **`pollAndFireWorkflowFollowups()`** — called every **10 seconds** from `index.ts` (both on `api` and `worker` roles). Uses `ZRANGEBYSCORE` (limit 1000/tick) to find due jobs, then `ZREM`s each one individually before enqueuing — the `ZREM` return value (0 or 1) is the **claim mechanism**: if multiple processes poll concurrently, only whichever one successfully removes the entry gets to enqueue it, preventing a double-fire. Claimed jobs are hand off to BullMQ with **`delay:0`** (goes straight to the `waiting` list, sidestepping the unreliable delayed-job promotion entirely). If the BullMQ enqueue itself fails, the job is re-added to the Redis sorted set with a 60-second future score, so it's retried on a later poll rather than lost.

## 6. Follow-up execution (`jobs/workflowFollowup.queue.ts` + `.worker.ts`)

The BullMQ `workflow-followup` queue/worker is the final execution step: `startWorkflowFollowupWorker()` runs at **concurrency 50** with a rate limiter of **50 jobs/sec** (matched to Twilio's per-account concurrent-call ceiling). Each job simply calls `call.service.dispatchCall(userId, agentId, to, 'outbound', contactId, /*workflowFollowup*/ true)` — the `workflowFollowup:true` flag is what makes `dispatchCall` (see `03-calls-realtime-pipeline.md §3`) refuse to fall back to any platform/env caller-ID and instead require a customer-owned number, and what marks the resulting `Call.workflowFollowup:true` for the loop-prevention checks described above. Job failures are logged and re-thrown so BullMQ's own failure tracking picks them up (no custom retry logic here — whatever BullMQ's default job options provide).

## 7. Time-based (non-event) triggers (`workflowScheduler.service.ts` + `jobs/workflowScheduler.queue.ts`/`.worker.ts`)

Workflows whose trigger is `scheduled` (a fixed daily/weekly time-of-day call, not tied to any call/lead event) are handled entirely separately from §4's event-trigger path:

- `jobs/workflowScheduler.worker.ts` registers a **repeatable 1-minute BullMQ job** (`startSchedulerTick`) whose handler is `fireScheduledWorkflows()`.
- Every tick: loads **every** `active` workflow platform-wide (not scoped to one user — this is a global sweep), filters to `triggerType === 'scheduled'`, and for each one:
  - **Once-per-calendar-day guard**: compares `lastExecutedAt` against "now" **in the workflow's own configured timezone** (via `toLocaleDateString('en-CA', {timeZone: tz})` → `YYYY-MM-DD` string comparison) — skips if already fired today in that timezone.
  - **Time match with catch-up window**: computes current `hour:minute` in the workflow's timezone, compares against the configured `scheduledTime`, and fires if the current time is **0–5 minutes past** the target (a wrap-safe modulo diff) — this deliberately tolerant window exists so a server restart or a slightly-late tick doesn't cause a scheduled call to be silently skipped for the entire day.
  - **Frequency filter**: `weekdays` (Mon–Fri only) or `weekly` (only on the explicitly listed day names) narrow which days it's allowed to fire on; `daily` has no restriction.
  - Walks the remaining nodes (same delay-accumulation logic as §4, duplicated here rather than shared) and schedules each `call` node via the same `scheduleWorkflowFollowupJob` used by the event-trigger path — so from §5 onward, scheduled and event-triggered workflows converge onto the identical execution mechanism.
  - Updates `executions`/`lastExecutedAt` on success.

## 8. Rebuild Notes (Replit target)

### No billing coupling
This domain has no dependency on the billing/subscription system directly — the one gate it does enforce (`assertCustomerOwnedNumberForAutomations`) is a **caller-ID/Twilio-ownership** policy, not a plan/quota check, and should be kept regardless of whether billing is stripped elsewhere (it exists to guarantee automation calls always go out under the tenant's own number, not a shared platform number — a correctness concern, not a monetization one).

### Dependencies
`mongoose`, `ioredis` (via `config/redis.ts`, for the sorted-set poller), `bullmq`. No third-party workflow/automation engine — this is fully hand-rolled.

### Consolidation opportunity (not required, but worth doing during a rebuild)
The `config: Map<string,string>` reading helpers (`cfgMap`/`cfgGet`/`extractConfig`/`cfgVal`) are reimplemented with slightly different signatures in `workflow.service.ts`, `workflowRunner.service.ts`, and `workflowScheduler.service.ts`. Same for the preset-delay-minutes lookup table (duplicated verbatim in `workflowRunner.service.ts` and `workflowScheduler.service.ts`). A rebuild is a natural point to extract these into one shared `workflowConfig.util.ts`.

### Operational dependencies to preserve
- The **10-second Redis-sorted-set poll** (`pollAndFireWorkflowFollowups`, wired in `index.ts`) and the **1-minute scheduled-workflow BullMQ repeatable job** (`fireScheduledWorkflows`) are both independent of the request/response cycle — they must be started as part of process bootstrap (see `11-app-bootstrap-infra.md`) regardless of deployment target. On Replit, confirm the process that runs these intervals stays alive continuously (not spun down between requests) — this rules out a purely serverless/function-per-request deployment model for the API process.
- If billing/subscription plan tiers are fully removed, double-check nothing else implicitly depended on `Subscription` for workflow-specific limits — a grep of `workflow*.ts` during this doc's research turned up none, but re-verify against the live code at rebuild time.
