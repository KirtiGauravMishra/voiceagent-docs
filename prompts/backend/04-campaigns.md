# Backend Prompt 04 — Campaigns (Outbound Dialing)

> Read `../00-master-rebuild-guide.md`, `01-auth.md`, `02-agents.md`, `03-calls-realtime-pipeline.md` first.

## Goal
Bulk-dial a list of contacts with one agent and one caller-ID number, with pacing, retries, and a call-window schedule.

## Data models

**`Campaign`**: `ownerId`, `name` (max 200), `agentId` (→Agent, required), `phoneNumberId` (→PhoneNumber, required), `status` (enum `draft|scheduled|running|paused|completed|failed`), counters `totalContacts`/`contactedCount`/`completedCount`/`failedCount`/`skippedCount`, `scheduledAt`/`startedAt`/`completedAt`, `notes` (max 1000), and embedded `settings`: `maxConcurrentCalls` (1–50, default 1), `callIntervalSeconds` (default 5), `maxAttempts` (1–5, default 2), `retryIntervalMinutes` (min 5, default 60), `timezone` (default `'UTC'`), `callWindowStart`/`callWindowEnd` (`HH:MM`, default `00:00`/`23:59`).

**`CampaignContact`** (one row per contact in a campaign): `campaignId`, `ownerId`, `contactId` (optional link to the Contact collection), `name`, `phone`, `status` (enum `pending|calling|completed|failed|no-answer|busy|voicemail|skipped`), `attempts`, `callId` (→Call, most recent), `lastAttemptAt`/`nextAttemptAt`.

Index `{campaignId,status}` and `{campaignId,nextAttemptAt,status}` (powers "give me the next due batch").

## Creating a campaign

Input: `{name, agentId, phoneNumberId?, scheduledAt?, notes?, settings?, contacts:[{name,phone,contactId?}]}`. If `phoneNumberId` omitted, resolve: owner's default active number → any active owner number → error if none (no shared-pool fallback needed since this is a single-org tool). Verify the agent exists, is owned, and is active. Set `status` to `scheduled` if `scheduledAt` given, else `draft`. Bulk-insert `CampaignContact` rows (`insertMany` with `ordered:false` so one bad row doesn't abort the batch).

## Launch / pause / resume / delete

- **Launch**: reject if already `running`/`completed`, reject if zero `pending` contacts. Set `status:'running'`, enqueue a `run-campaign` BullMQ job.
- **Pause**: only from `running` → `paused`.
- **Resume**: only from `paused` → `running`, re-enqueue.
- **Delete**: reject while `running` (must pause first). Delete the campaign, all its `CampaignContact` rows, and any queued job.

**Enqueue resilience**: if the BullMQ enqueue call itself fails transiently (Redis blip), don't roll the campaign back to `draft` — leave it `running` and let the sweeper (below) pick it up within seconds. Only roll back on a non-transient failure.

## The dialing worker (BullMQ, one job per campaign launch/resume)

Loop until done, paused, or errored:
1. Re-check the campaign's live status from the DB; stop if no longer `running`.
2. Check the call-window (current time in the campaign's timezone, inside `callWindowStart`–`callWindowEnd`); if outside, sleep ~60s and recheck rather than erroring.
3. Select the next batch: `CampaignContact` where `status:'pending'` and (`nextAttemptAt` unset or due), limited to `maxConcurrentCalls`. If empty and nothing is currently `calling`, the campaign is done — mark `completed`.
4. **Claim the batch first** (`updateMany` → `status:'calling'`, bump `attempts`) before dialing, so a slow dial can't be double-picked.
5. Dial each contact in the batch **in parallel**: validate/normalize to E.164 (10-digit → assume this org's primary country code — adjust if different; anything unparseable → `status:'skipped'`). Create a `Call` row (`campaignId`/`campaignContactId` set) and place it through the same call-dispatch path as Prompt 03. On dial failure: retry later (`status:'pending'`, `nextAttemptAt = now + retryIntervalMinutes`) up to `maxAttempts`, then `status:'failed'`.
6. Recompute the campaign's aggregate counters (`$group` by status) and update the campaign doc; mark `completed` if nothing is `pending` anymore.
7. Sleep `callIntervalSeconds` before the next batch.

**Sweeper** (independent `setInterval`, every ~10s, not tied to BullMQ): directly queries Mongo for any `running` or due-`scheduled` campaign not already being processed in this instance, and starts the dialing loop for it. This guarantees forward progress even if a BullMQ enqueue was silently lost — a genuine safety net, keep it.

**Concurrency cap**: don't gate `maxConcurrentCalls` behind any plan/subscription lookup — there's no plan system in this rebuild. Either use the campaign's own configured `maxConcurrentCalls` directly, or (if you want an org-wide safety ceiling to avoid exceeding your Twilio account's concurrent-call limit) apply one flat env-var-driven cap, not a per-plan one.

## API — `/api/campaigns` (JWT)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | paginated list, populate agent name/phone number |
| POST | `/` | create |
| GET | `/:id` | detail |
| GET | `/:id/contacts` | paginated per-campaign contact list, populate call status/duration |
| POST | `/:id/launch` | start |
| POST | `/:id/pause` / `/:id/resume` | pause/resume |
| DELETE | `/:id` | delete (blocked while running) |

## Environment variables
`TWILIO_PHONE_NUMBER` (fallback default caller ID only), `REDIS_URL` (BullMQ).

## Acceptance checklist
- [ ] Launching a campaign with 3+ contacts dials them respecting `maxConcurrentCalls` and `callIntervalSeconds`.
- [ ] A contact with an invalid phone number is marked `skipped`, not retried.
- [ ] A failed dial retries up to `maxAttempts` with the configured backoff, then gives up.
- [ ] Pausing mid-run stops new dials but doesn't corrupt in-flight ones; resuming continues where it left off.
- [ ] If Redis briefly drops during launch, the campaign still starts dialing within ~10s via the sweeper.
- [ ] Deleting a running campaign is rejected; pausing first then deleting works.
