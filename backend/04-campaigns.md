# 04 — Campaigns (Outbound Dialing)

> Backend domain doc. Source: `backend/src/{models/Campaign.model.ts, controllers/campaign.controller.ts, routes/campaign.routes.ts, services/campaign.service.ts, jobs/campaign.queue.ts, jobs/campaign.worker.ts}`.

## 1. Overview

A **Campaign** bulk-dials a list of contacts with one agent, one caller-ID phone number, and a set of pacing/retry rules. It's the "call 500 leads automatically" feature, distinct from `Workflow` (event-triggered single automation calls, see `06-workflows.md`) and from ad-hoc manual `Call` dispatch (`03-calls-realtime-pipeline.md`) — though all three ultimately call the same `initiateOutboundCall()` Twilio integration.

Two collections: `Campaign` (the container — settings, status, aggregate counters) and `CampaignContact` (one row per contact in the campaign — its own dial status, attempt count, and the `Call` it produced).

## 2. Data Model (`models/Campaign.model.ts`)

### `Campaign`
| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `name` | string | max 200 |
| `agentId` | ObjectId → Agent | required |
| `phoneNumberId` | ObjectId → PhoneNumber | required — the caller ID used for every call in the campaign |
| `status` | enum | `draft → scheduled → running → paused → completed \| failed` |
| `totalContacts` / `contactedCount` / `completedCount` / `failedCount` / `skippedCount` | number | aggregate progress counters, recomputed after each dial batch |
| `scheduledAt` / `startedAt` / `completedAt` | Date | |
| `settings` | embedded | see below |
| `notes` | string | max 1000 |

### `settings` (embedded, `_id:false`)
| Field | Default | Notes |
|---|---|---|
| `maxConcurrentCalls` | `1` | 1–50 — parallel calls per dial batch, further capped by the owner's `Subscription.maxConcurrentCalls` at runtime (billing-plan concept — see rebuild note) |
| `callIntervalSeconds` | `5` | pause between batches |
| `maxAttempts` | `2` | 1–5, per-contact retry cap |
| `retryIntervalMinutes` | `60` | min 5 — backoff before retrying a failed contact |
| `timezone` | `'UTC'` | IANA tz, used to evaluate the call window |
| `callWindowStart` / `callWindowEnd` | `'00:00'` / `'23:59'` | `HH:MM` 24h — calls only placed inside this window in the campaign's timezone |

### `CampaignContact`
| Field | Type | Notes |
|---|---|---|
| `campaignId` | ObjectId → Campaign | indexed |
| `ownerId` | ObjectId → User | indexed |
| `contactId` | ObjectId → Contact | optional link to the CRM contact |
| `name` / `phone` | string | denormalized at import time — a campaign contact doesn't have to originate from the `Contact` collection |
| `status` | enum | `pending \| calling \| completed \| failed \| no-answer \| busy \| voicemail \| skipped` |
| `attempts` | number | incremented each dial attempt |
| `callId` | ObjectId → Call | the most recent `Call` produced for this contact |
| `lastAttemptAt` / `nextAttemptAt` | Date | `nextAttemptAt` drives retry scheduling |

Indexes: `{campaignId:1, status:1}` and `{campaignId:1, nextAttemptAt:1, status:1}` — both power the dialer's "give me the next pending batch that's due" query.

## 3. API Reference — `/api/campaigns` (JWT required)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list, populates `agentId`(name/status) and `phoneNumberId`(number/friendlyName) |
| POST | `/` | Create (see §4) |
| GET | `/:id` | Detail |
| GET | `/:id/contacts` | Paginated contact list for one campaign, populates `callId` (status/duration/sid) |
| POST | `/:id/launch` | Start dialing (see §5) |
| POST | `/:id/pause` | `running → paused` (no-op/404 unless currently `running`) |
| POST | `/:id/resume` | `paused → running`, re-enqueues the dialer job |
| DELETE | `/:id` | Delete campaign + all its `CampaignContact` rows; blocked while `running` |

## 4. Creating a campaign (`campaign.service.createCampaign`)

Input: `{ name, agentId, phoneNumberId?, scheduledAt?, notes?, settings?, contacts: [{name, phone, contactId?}] }`.

**Phone number resolution** (only if `phoneNumberId` omitted): owner's default active number → any active owner number → any active company (platform pool) number → **auto-seed** a `PhoneNumber` row from the `TWILIO_PHONE_NUMBER` env var if nothing else exists and that env var is set (marks it `isCompany:true, isDefault:true`). If nothing resolves at all, `400`.

Agent must exist, be owned by the caller, and be `status:'active'` (404 otherwise).

`status` is set to `'scheduled'` if `scheduledAt` was given, else `'draft'`. Contacts are bulk-inserted into `CampaignContact` with `insertMany(..., {ordered:false})` (a failure on one row doesn't abort the rest).

## 5. Launch / Resume — enqueue with DB-sweeper fallback

`launchCampaign`/`resumeCampaign` both: validate state (can't launch an already-`running`/`completed` campaign; must have ≥1 `pending` contact to launch), quota-check (`assertCanPlaceVoiceCall` — billing-driven, see rebuild note), flip `status` to `running`, then enqueue a `run-campaign` BullMQ job.

**If the enqueue itself fails** (checked against a list of known-transient Redis error substrings: `Connection is closed`, `ECONNRESET`, `Command timed out`, `connecting/connected`): the campaign is **not** rolled back to `draft`/`paused` — instead the status stays `running` and a background **sweeper** (see §7) picks it up within 10 seconds regardless. Only a **non**-transient enqueue error rolls the status back and surfaces a `503` to the caller. This two-tier handling exists so a momentary Redis blip during launch doesn't strand a campaign in a broken state or force the user to retry.

## 6. The dialing loop (`campaign.worker.ts::runCampaign`)

One `run-campaign` job triggers this whole loop (it doesn't return until the campaign is fully done, paused, or errors out) — `activeCampaigns` (an in-process `Set<campaignId>`) prevents two invocations (e.g. the queue job *and* the sweeper) from double-driving the same campaign concurrently.

Per iteration:
1. **Re-check status from DB** — if no longer `running` (paused, deleted, etc.), stop the loop.
2. **Quota check** (`assertCanPlaceVoiceCall`) — on failure, the *whole campaign* auto-pauses (`status:'paused'`) rather than erroring the job, so it's resumable later once quota is available again.
3. **Call-window check** (`isWithinCallWindow`) — if outside the configured window (or window invalid → fails open, doesn't block), sleeps 60s and re-checks rather than erroring or advancing.
4. **Batch selection**: `CampaignContact.find({campaignId, status:'pending', $or:[{nextAttemptAt: !exists}, {nextAttemptAt: <= now}]}).limit(concurrency)` — `concurrency = min(settings.maxConcurrentCalls, subscription.maxConcurrentCalls)` (the plan-tier cap; **billing-coupled**, see rebuild note). If the batch is empty: if there are still contacts `calling` (in flight), wait 3s and recheck; otherwise the campaign is done, loop ends.
5. **Claim the batch**: `updateMany` sets `status:'calling'`, bumps `attempts`, stamps `lastAttemptAt` — claimed *before* dialing so a slow dial can't be picked up twice.
6. **Dial each contact in the batch in parallel** (`Promise.allSettled`):
   - Validates/normalizes phone to E.164 (`toE164` — handles bare 10-digit US numbers, 11-digit with leading `1`, or any 9–15 digit international number; anything else → `status:'skipped'`, `skippedCount++`).
   - Creates a `Call` row (`campaignId`/`campaignContactId` set, `status:'queued'`), links it back onto the `CampaignContact.callId`, then calls the **same** `initiateOutboundCall()` used by manual dispatch (`03-calls-realtime-pipeline.md`).
   - On dial failure: if `attempts >= maxAttempts`, marks the contact `failed` permanently; otherwise resets it to `pending` with `nextAttemptAt = now + retryIntervalMinutes` for a later retry pass.
7. **`updateCampaignProgress(campaignId)`** — aggregates `CampaignContact` counts by status (`$group`), updates the campaign's counters, and if `pending === 0`, marks the campaign `completed` (`completedAt` stamped) and ends the loop.
8. Sleep `callIntervalSeconds` before the next batch.

Worker concurrency: up to **10 campaigns** run their `runCampaign` loops simultaneously (BullMQ worker `concurrency:10`); each campaign internally fans out up to `maxConcurrentCalls` (≤50) calls per batch.

## 7. The sweeper (fallback safety net)

Independent of BullMQ, `campaign.worker.ts` also runs a `setInterval` every **10 seconds** that directly queries Mongo for any `Campaign` with `status:'running'` or (`status:'scheduled'` and `scheduledAt <= now`), and — for anything not already in `activeCampaigns` — starts `runCampaign()` on it directly (flipping `scheduled → running` first if needed). This exists specifically so that **API→Redis enqueue failures never permanently strand a campaign**: even if `addCampaignJob` never successfully queued a job, the sweeper will pick up any campaign sitting in `running`/due-`scheduled` state within 10 seconds, guaranteeing forward progress independent of Redis/BullMQ health.

## 8. Queue plumbing (`campaign.queue.ts`)

`addCampaignJob`/`removeCampaignJob` each open a **fresh, short-lived Redis connection** per call (via `makeBullmqQueueConnection()`) rather than reusing a shared pooled connection — explicitly so a stuck/half-dead shared connection can never block a launch. `addCampaignJob` retries transient errors (same substring list as above) up to 3 times with linear backoff (300ms × attempt) before giving up; `removeCampaignJob` (used on delete) swallows a "job not found" error silently. Default job options: `attempts:1` (the campaign loop itself handles retries at the contact level, not BullMQ's job-level retry), `removeOnComplete/Fail: {count:50}` (keeps the last 50 job records for debugging, auto-trims older ones).

## 9. Rebuild Notes (Replit target, in-house/no-billing)

### Billing coupling to address
- `campaign.service.launchCampaign`/`resumeCampaign` call `assertCanPlaceVoiceCall(ownerId)` — same quota gate documented in `03-calls-realtime-pipeline.md`; remove or stub for an in-house build with no metered billing.
- `campaign.worker.ts::runCampaign` looks up `Subscription.maxConcurrentCalls` to cap the effective per-batch concurrency (`concurrency = min(settings.maxConcurrentCalls, planLimit)`, defaulting to `1` if no subscription found — meaning **today, with billing removed and no Subscription documents, every campaign would silently run at concurrency 1** regardless of its own `settings.maxConcurrentCalls`). **This needs an explicit decision during rebuild**: either delete the `Subscription` lookup entirely and use only `settings.maxConcurrentCalls` (recommended for an in-house tool with a fixed calling capacity), or replace it with a flat env-var-driven org-wide concurrency cap if you still want a global ceiling (e.g. to protect against exceeding your Twilio account's concurrent-call limit).
- Also re-checks `assertCanPlaceVoiceCall` every dial-loop iteration (not just at launch) — same removal applies.

### Dependencies
`bullmq`, `ioredis` (via `config/redis.ts`'s `makeBullmqQueueConnection`), `mongoose`. No third-party campaign/dialer SaaS — this is a hand-rolled dialer built directly on Twilio's Calls API.

### Env vars relevant to this domain
- `TWILIO_PHONE_NUMBER` — used only as the auto-seed fallback when a campaign is created with no phone number configured anywhere.
- `REDIS_URL` — required for the BullMQ queue; note the sweeper means the feature *degrades gracefully* (10s-latency dialing instead of instant) rather than fully breaking if Redis has a bad moment, but Redis is still a hard dependency for the queue itself to exist.

### Operational notes to carry into the rebuild
- The call-window/timezone check uses `Intl.DateTimeFormat` (no `moment`/`date-fns-tz` dependency) — keep this pattern, it's already dependency-free.
- The E.164 normalizer (`toE164`) assumes **10-digit numbers are US** (prepends `+1`) — if the in-house rebuild targets a different primary country, this default should change.
- `activeCampaigns` (in-process `Set`) means campaign-loop de-duplication **only works within a single worker process** — if you ever run multiple worker instances, both the queue's own `jobId` uniqueness (`campaign-${id}-${timestamp}` — note this is NOT idempotent since it includes a timestamp, so it doesn't actually prevent duplicate `run-campaign` jobs for the same campaign from BullMQ's perspective) and the sweeper could both spin up a second concurrent `runCampaign` for the same campaign on a different instance. Fine for a single-instance Replit deployment; revisit if scaling out.
