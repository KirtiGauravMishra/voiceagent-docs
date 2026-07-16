# 09 — Background Jobs & Queues

> Backend domain doc. Source: `backend/src/{jobs/*.queue.ts, jobs/*.worker.ts, config/redis.ts}`. Covers the main-product job system (call simulation, campaign dialer, workflow follow-ups, workflow scheduler). The `leadSequence` queue/worker (drip-email sends for the internal sales CRM) is out of scope per current project priorities — see `05-leads-contacts-crm.md` if that system is revisited later.

## 1. Overview

All background work runs on **BullMQ** (Redis-backed job queues). The process that runs workers is controlled by the `ROLE` env var (`api | worker | all`, see `11-app-bootstrap-infra.md`) — on a single small deployment (the expected Replit target), `ROLE=all` runs everything in one process; at larger scale the API and worker processes could be split across separate deployments sharing the same Redis instance.

Four BullMQ queue/worker pairs exist in the main product surface:

| Queue name | File pair | Purpose | Concurrency |
|---|---|---|---|
| `call-processing` | `call.queue.ts` / `call.worker.ts` | Dev-only call **simulation** fallback | 5 (rate-limited 20/sec) |
| `campaign-dialer` | `campaign.queue.ts` / `campaign.worker.ts` | Outbound campaign dialing loop | 10 (each campaign internally fans out ≤50 calls) |
| `workflow-followup` | `workflowFollowup.queue.ts` / `workflowFollowup.worker.ts` | Executes scheduled automation follow-up calls | 50 (rate-limited 50/sec) |
| `workflow-scheduler` | `workflowScheduler.queue.ts` / `workflowScheduler.worker.ts` | Drives the 1-minute "check for due scheduled workflows" tick | 1 |

Full behavioral detail for the campaign dialer is in `04-campaigns.md §6-8`; full detail for workflow follow-up scheduling/execution and the scheduler tick is in `06-workflows.md §5-7`. This file focuses on the **shared Redis/BullMQ plumbing** and fills in the `call-processing` queue (not covered elsewhere).

## 2. Redis connection strategy (`config/redis.ts`)

Three distinct Redis client roles are deliberately kept separate, each tuned for its own failure mode:

1. **`bullmqRedis`** — the single long-lived connection **Workers** use to block on `BRPOP`-style commands waiting for jobs. Configured with `maxRetriesPerRequest: null` (**required** by BullMQ — a normal retry limit would break the indefinite blocking wait) and `enableReadyCheck: false`. A `retryStrategy` backs off linearly up to 5s. Logs are deliberately quiet about routine connection blips (`ECONNRESET`/`EPIPE`/etc. logged at `debug`, not `error`) since Redis providers (especially free/serverless tiers) drop idle connections routinely and BullMQ auto-reconnects — only a genuine `ENOTFOUND` (wrong hostname — e.g. a Redis instance was deleted/rotated and `REDIS_URL` wasn't updated) is logged once at `warn` (deduped via a module-level flag so it doesn't spam on every reconnect attempt).
2. **`makeBullmqQueueConnection()`** — a **factory**, not a singleton. Every `Queue.add()` call site (`addCampaignJob`, `addCallJob`, `addWorkflowFollowupJob`) creates a **fresh, short-lived connection**, uses it once, then explicitly closes and disconnects it. The comment in the source explains why: some serverless/free-tier Redis providers silently drop idle connections, and a long-lived shared queue-side connection can end up "connected" from Node's perspective but dead on the server side, causing enqueue calls to hang or fail unpredictably. A fresh connection per enqueue trades a small amount of latency/overhead for much higher reliability. `lazyConnect: true` + a short 5s connect timeout + a capped retry strategy (max 3 attempts) keep a bad connection attempt from hanging a request for long.
3. **`redis`** (shared client) — used for everything that **isn't** BullMQ: the rate limiter, the KB query-embedding cache, the workflow-followup Redis sorted set. Tuned for fail-fast behavior (`maxRetriesPerRequest: 2`, `commandTimeout: 2_000`) since none of these use cases should ever block a request indefinitely waiting on Redis — better to fail fast and fall back/error than hang.

`connectRedis()` (called once at boot from `index.ts`) pings both `redis` and `bullmqRedis` in parallel with a 5s timeout each, logging round-trip time — purely a startup diagnostic, not a hard gate (the app doesn't refuse to start if Redis is briefly slow to respond).

### Common resilience pattern across all three "add job" functions
`addCallJob`, `addCampaignJob`, `addWorkflowFollowupJob` all follow the identical shape: try up to 3 times, checking each failure against a shared list of "transient" error-message substrings (`ECONNRESET`, `Connection is closed`, `Command timed out`, `ETIMEDOUT`), sleeping `attempt * 300ms` between retries, and **always** closing the queue + disconnecting the connection in a `finally` block regardless of outcome. A non-transient error is re-thrown immediately (no point retrying a real validation/logic error). This pattern is copy-pasted across three files rather than shared — a natural consolidation target for a rebuild (e.g. a single `withBullmqQueue(name, fn)` helper).

## 3. `call-processing` queue — dev-mode call simulation (`call.queue.ts` / `call.worker.ts`)

**Not used in any real deployment with Twilio configured** — this is purely the fallback path `call.service.dispatchCall` takes when `TWILIO_ACCOUNT_SID`/`TWILIO_AUTH_TOKEN`/`TWILIO_WEBHOOK_BASE_URL` aren't all set (see `03-calls-realtime-pipeline.md §3`), letting a developer exercise the full Call/Agent/UI flow without any real telephony credentials.

- **Job options**: `attempts:3`, exponential backoff starting at 3s, keeps the last 100 completed / 200 failed job records.
- **Worker** (concurrency 5, rate-limited to 20/sec): flips the `Call` to a (non-schema!) `status:'processing'` value, sleeps a random 2–5 seconds to simulate call duration, then writes a canned fake transcript (`"[OUTBOUND] Agent "alloy" responded to +1555...\nSystem prompt applied. Call completed successfully."`) and a random 30–150 second `durationSeconds`, marking the call `completed`.
- On permanent failure (all 3 attempts exhausted): marks the `Call` `status:'failed'` with the error message as `failedReason`.

> Note: `status:'processing'` is set mid-simulation but **is not one of the enum values** on `Call.model.ts`'s `status` field (`queued|ringing|in-progress|completed|failed|no-answer|busy|voicemail`) — Mongoose will reject this write against the real schema's enum validator unless the model's strictness allows it, or this is simply a latent bug that's never been hit because this dev-only path is rarely exercised with schema validation strictly enforced. Worth fixing (`'processing'` → `'in-progress'`) during a rebuild rather than carrying the inconsistency forward.

## 4. Campaign dialer, Workflow follow-ups, Workflow scheduler

Fully documented in their respective domain files:
- **`campaign-dialer`** — `04-campaigns.md §6` (the dialing loop itself), `§7` (the 10-second DB-sweeper fallback), `§8` (queue plumbing specifics).
- **`workflow-followup`** — `06-workflows.md §5` (the Redis-sorted-set → BullMQ two-layer scheduling design and why), `§6` (the worker itself — just calls `dispatchCall(..., workflowFollowup:true)`).
- **`workflow-scheduler`** — `06-workflows.md §7`. One implementation detail not covered there: `startWorkflowSchedulerWorker()` (`jobs/workflowScheduler.worker.ts`) first calls `startSchedulerTick()` (`jobs/workflowScheduler.queue.ts`, not separately detailed elsewhere) to **register a BullMQ repeatable job** — this is BullMQ's built-in cron-like repeat-job feature (not the Redis-sorted-set pattern used for follow-ups), configured for a 1-minute interval. If registering the repeatable job fails (e.g. Redis unavailable at boot), it's logged as a warning and the scheduler is effectively disabled until the next process restart — there's no independent retry-to-register mechanism.

## 5. Rebuild Notes (Replit target, in-house/no-billing)

### No billing coupling in the queue/worker plumbing itself
The queues/connection layer documented here has no billing dependency. (The campaign worker's `Subscription.maxConcurrentCalls` lookup is a billing concern documented and flagged in `04-campaigns.md §9` — not part of this file's scope.)

### Dependencies
`bullmq`, `ioredis`. No other third-party job-queue service.

### Env vars
`REDIS_URL` — the only one specific to this layer (already covered in other docs' env tables; repeated here for completeness since every queue depends on it). `stripToRedisUri()` (`config/env.ts`) defensively sanitizes this value — worth knowing if you ever paste a Redis connection string from a provider's dashboard that includes CLI-wrapper syntax (e.g. `redis-cli -u redis://...`) or URL-encoded artifacts; the sanitizer strips those down to a bare `redis://`/`rediss://` URI before it reaches `ioredis`.

### Operational requirements for Replit specifically
- **The process must stay running continuously**, not spin up per-request — BullMQ workers block waiting on Redis, the workflow-followup poller runs a 10-second `setInterval`, and the workflow-scheduler's repeatable job needs a live worker process to pick it up every minute. Confirm whatever Replit deployment mode is chosen (Reserved VM / "Always On" style, not an ephemeral/serverless function) keeps one process alive indefinitely. This is the same constraint noted in `06-workflows.md §8`.
- **Single-instance is fine, multi-instance needs care**: the short-lived-connection-per-enqueue pattern and the transient-error-retry logic are resilience features for a *flaky Redis provider*, not for *concurrent worker instances* — nothing here provides distributed locking beyond what's already noted per-domain (the campaign worker's in-process `activeCampaigns` Set, the workflow poller's Redis `ZREM`-as-claim pattern). If the in-house rebuild only ever runs one Replit instance (likely, given the "no billing / in-house" scope), this is a non-issue.
- If you provision a fresh Redis instance for the rebuild (e.g. Replit's Redis integration, or an external provider like Upstash), just set `REDIS_URL` — no other config in this layer needs to change.

### Bug to fix during rebuild
Fix the `status:'processing'` vs. schema-enum `'in-progress'` mismatch in `call.worker.ts` (§3) — cosmetic today only because this dev-simulation path is rarely exercised, but worth correcting so the simulated path exactly mirrors real-call status semantics.
