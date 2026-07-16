# Backend Prompt 06 — Workflows (Call Automation)

> Read `../00-master-rebuild-guide.md`, `02-agents.md`, `03-calls-realtime-pipeline.md`, `05-leads-contacts-crm.md` first.

## Goal
Simple "when X happens, wait N minutes, then call this number with this agent" automation — a linear chain of `trigger → delay → call` nodes. Not a general graph/flowchart engine — one straight line per workflow.

## Data model — `Workflow`

`ownerId`, `name`, `status` (enum `active|inactive`), `nodes` (array — see shape below), `executions` (count), `lastExecutedAt`.

**Node shape**: `{id, type: 'trigger'|'delay'|'call', title, subtitle, config: Record<string,string>}`. `nodes[0]` must always be type `trigger`. Store `config` as a plain string-keyed map (if using Mongoose, a `Map<String,String>` works — just be consistent about how you read it, since `Map` doesn't support bracket-notation access).

**Trigger config** (`config.triggerType`): `call_completed | call_started | inbound_call | lead_created | scheduled`. For `scheduled`, also store `scheduledPhone`, `scheduledTime` (`HH:MM`), `scheduledTimezone` (IANA, default e.g. your org's primary timezone), `scheduledFrequency` (`daily|weekdays|weekly`), `scheduledDays` (comma-separated day names, for `weekly`).

**Delay config** (`config.delayType`): one of a fixed preset list (`5min,15min,30min,1hour,2hours,3hours,6hours,12hours,24hours,48hours,72hours`), or `custom` (+`delayCustomMinutes`, clamp 1–10080 i.e. max 7 days), or `user_requested` (resolved at trigger-time from an actual callback datetime supplied by the trigger context, e.g. "call me back at 3pm" — if no such datetime is available when the trigger fires, abort scheduling for that firing rather than guessing a default).

**Call config**: `agentId` (+ denormalized `agentLabel` for display).

## Validation & activation rules
- Reject `nodes` that don't start with a `trigger` node.
- Require a customer-owned (non-shared) phone number to exist before allowing `status:'active'` — automation calls must go out under this org's own number (same rule as the call-dispatch caller-ID policy in Prompt 03).
- Require at least one `call` node with an `agentId` set before allowing activation.
- For `scheduled` triggers, require both `scheduledPhone` and `scheduledTime` before allowing activation.

## Trigger firing

Two firing entry points, called from wherever the actual event happens elsewhere in the codebase:
- **Event-triggered** (`call_completed`, `call_started`, `inbound_call`, `lead_created`): call this whenever that event actually occurs (from the call-status webhook handler, the inbound-call handler, and lead creation). **Guard against infinite loops**: skip firing if the call that just completed was itself a workflow-triggered follow-up call (check a `workflowFollowup` flag on the Call).
- For each `active` workflow whose trigger type matches: walk the node list in order, accumulating delay minutes across `delay` nodes, and for each `call` node schedule a delayed job using the accumulated delay-so-far. If a `user_requested` delay has no resolvable callback time, abort scheduling for that entire workflow firing (don't schedule any of its call nodes). On any successful scheduling, increment `executions` and stamp `lastExecutedAt`.

## Scheduling mechanism (don't use BullMQ's native delayed jobs directly — see why)

Use a **Redis sorted set** as the durable timer: `ZADD` a JSON job payload scored by its fire-at timestamp. A separate poller (every ~10s) does `ZRANGEBYSCORE` for due jobs, `ZREM`s each one (the `ZREM` return value is your at-most-once claim — if it returns 0, another poller instance already claimed it), then hands it to a BullMQ queue with **zero delay** (goes straight to the `waiting` list). This two-layer design avoids known reliability issues with BullMQ's own delayed-job promotion mechanism on some managed Redis providers. The actual follow-up call execution worker (concurrency ~50, rate-limited to match your telephony provider's per-account call-placement rate) just calls the same call-dispatch function from Prompt 03 with a `workflowFollowup:true` flag.

## Time-based (non-event) triggers

A separate, simpler mechanism for `scheduled`-type triggers: a repeating **1-minute tick** (BullMQ repeatable job or a plain `setInterval` in the worker process) that scans every `active` workflow with a `scheduled` trigger, and for each one:
- Skip if it already fired today (compare "today" in the workflow's own timezone against `lastExecutedAt`, not server time).
- Fire if the current time in the workflow's timezone is within a few minutes past the target time (a small tolerance window — e.g. 0–5 minutes — so a slightly-late tick or a brief restart doesn't cause a whole day to be skipped).
- Respect the frequency filter (`daily`/`weekdays`/specific `weekly` days).
- Walk the same delay-accumulation + call-scheduling logic as the event-triggered path, converging on the same follow-up scheduling mechanism above.

## API — `/api/workflows` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | paginated, filter `search`/`status` |
| POST | `/` | create — `{name, agentId?}`; seed a default `call_completed` trigger node, append one `call` node if `agentId` given |
| GET | `/:id` | detail |
| PATCH | `/:id` | update `name`/`status`/`nodes` (whole-array replace on `nodes`, with the validation/activation rules above) |
| PATCH | `/:id/status` | convenience — same as PATCH with just `status` |
| DELETE | `/:id` | delete |

## Environment variables
`REDIS_URL` (required for both the sorted-set poller and BullMQ).

## Acceptance checklist
- [ ] A `call_completed`-trigger workflow with one delay + one call node fires the follow-up call at (roughly) the right offset after a real call completes.
- [ ] A follow-up call completing does **not** itself trigger another follow-up (loop guard works).
- [ ] A `scheduled`-trigger workflow fires once per day at the configured local time in its own timezone, not server time.
- [ ] Activating a workflow with no `call` node, or with a `scheduled` trigger missing phone/time, is rejected with a clear error.
- [ ] If the scheduling process restarts, previously-scheduled-but-not-yet-fired follow-ups still fire (durable in Redis, not lost).
