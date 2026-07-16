# Backend Prompt 09 — Background Jobs & Queues

> Read `../00-master-rebuild-guide.md`, `03-calls-realtime-pipeline.md`, `04-campaigns.md`, `06-workflows.md` first.

## Goal
Consolidate every background-processing concern into one consistent BullMQ setup. This prompt is the cross-cutting reference for the queues the earlier prompts already introduced individually — build the shared infrastructure once, here, then wire each queue's worker into its owning module.

## Why BullMQ (and why it must be multi-instance-safe)

Given the scale target (100k+ calls/day, 1,000+ concurrent), this rebuild **must** run more than one app instance behind a load balancer. That rules out any in-process job mechanism (`setTimeout`, in-memory arrays/maps of pending work) — a job queued on instance A must be processable by a worker running on instance B. BullMQ (Redis-backed) is the shared backbone; every "background work" concern in this system should go through it rather than inventing a bespoke mechanism, **except** the two durable-timer cases (workflow follow-ups, Prompt 06; and the call-completion stuck-call sweep, Prompt 03) which intentionally use a Redis sorted set in front of BullMQ for reasons explained in those prompts — don't relitigate that choice here, just keep it consistent.

## Redis connection

One shared ioredis connection (or a small connection-per-purpose pool if your BullMQ version recommends separate connections for the queue vs. the blocking worker vs. events) — read `REDIS_URL` once at startup, fail fast if unset or unreachable. Set `maxRetriesPerRequest: null` on the connection used for BullMQ (a documented BullMQ requirement — its blocking commands don't play well with ioredis's request retry limit).

## Queue inventory

Build these queues (names indicative — keep them consistent with whatever you called the corresponding job types in earlier prompts):

| Queue | Producer | Consumer concurrency | Purpose |
|---|---|---|---|
| `campaign-dial` | Campaign launch/resume (Prompt 04) | ~1 worker per running campaign's loop, but cap total concurrent campaign workers per instance | drives the per-campaign dialing loop |
| `workflow-followup` | Redis-sorted-set poller (Prompt 06) | ~50 | places the actual follow-up call |
| `workflow-scheduled-tick` | repeatable job, every 1 min | 1 | scans `scheduled`-trigger workflows |
| `knowledge-ingest` | Knowledge doc upload (Prompt 07) | ~10 (CPU/IO-bound text extraction + embedding calls) | chunk + embed a document |
| `post-call-processing` | Call completion (Prompt 03) | ~20 | transcript summarization, extraction, webhook delivery |

Add/rename queues as needed for whatever else earlier prompts implied, but keep this table as the single place a future maintainer looks to find "what background work exists in this system."

## Worker conventions

- **Idempotency**: every job handler must tolerate at-least-once delivery (BullMQ's default guarantee). Before doing external side effects (placing a call, sending an email), check whether the target record's state already reflects "this job already ran" and no-op if so.
- **Retries**: configure `attempts` (2–5 depending on the job's own cost/risk of retrying) with exponential `backoff`. Distinguish transient failures (network blip — let BullMQ retry) from permanent ones (invalid phone number — mark failed immediately, don't retry).
- **Concurrency**: set per-worker `concurrency` deliberately (table above) — don't default everything to 1 (underutilizes) or unbounded (can overwhelm downstream providers). Tie call-placement-related concurrency to your telephony provider's actual account-level concurrent-call ceiling.
- **Observability**: log job start/complete/fail with the job's queue name, id, and a correlation id (e.g. `callId`/`campaignId`) so a failure is traceable back to the business object it affected. Attach `queue.on('failed', ...)` / `worker.on('failed', ...)` handlers that log with enough context to debug without re-running the job.
- **Graceful shutdown**: on `SIGTERM`, call `worker.close()` (BullMQ waits for in-flight jobs to finish before exiting) before the process exits — don't `process.exit()` out from under an in-flight call-placement job.

## Environment variables
`REDIS_URL` (shared across every queue).

## Acceptance checklist
- [ ] Killing one app instance mid-job (SIGTERM) lets in-flight jobs finish, and undelivered queued jobs are picked up by a surviving instance.
- [ ] Running two app instances simultaneously does not double-process any single job (BullMQ's own locking handles this — verify it, don't assume it).
- [ ] A transient failure (e.g. simulated Redis blip) retries with backoff; a permanent failure (e.g. malformed input) fails immediately without wasting retry attempts.
- [ ] Every queue in the inventory table above has a real worker registered — none silently unconsumed.
