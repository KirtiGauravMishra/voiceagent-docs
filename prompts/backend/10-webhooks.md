# Backend Prompt 10 — Outbound Webhooks

> Read `../00-master-rebuild-guide.md`, `02-agents.md`, `03-calls-realtime-pipeline.md`, `09-jobs-queues.md` first.

## Goal
Let this org push events (call completed, extraction results, campaign completed, etc.) to their own external systems (a CRM, a Zapier/Make webhook, an internal endpoint) — outbound only. This is distinct from the *inbound* Twilio webhooks covered in Prompt 03, which are provider callbacks into this system, not this system calling out.

## Data model — `WebhookEndpoint`

`ownerId`, `url` (required, must be `https://` in production), `secret` (auto-generated on create, used for request signing — never returned again after creation, only a masked preview), `events` (string[] — subset of the supported event-type enum, e.g. `call.completed|call.failed|campaign.completed|workflow.executed|lead.created`), `status` (enum `active|disabled` — auto-disable after repeated consecutive failures, see below), `consecutiveFailures` (counter), `lastTriggeredAt`, `lastStatus` (`success|failure`).

## Data model — `WebhookDelivery` (audit log)

`webhookEndpointId`, `event`, `payload` (the exact JSON body sent), `responseStatus`, `responseBody` (truncate to a few KB — don't store unbounded external response bodies), `attempt`, `success` (bool), `durationMs`, `createdAt`. Cap retention (e.g. TTL index of 30-90 days) — this table grows fast at scale and isn't meant to be kept forever.

## Delivery mechanism

Route every outbound webhook fire through **one function** (mirrors the master guide's `sendMail()`/`uploadBuffer()` choke-point pattern): `deliverWebhook(ownerId, eventType, payload)`. That function:
1. Looks up all `active` `WebhookEndpoint`s for this owner subscribed to `eventType`.
2. For each, enqueues a `webhook-delivery` BullMQ job (don't call `fetch` synchronously inline in the request/call path — a slow or hanging external endpoint must never block call processing).
3. The worker (concurrency ~30, `attempts: 5` with exponential backoff, timeout ~10s per attempt) POSTs the JSON payload with headers: `Content-Type: application/json`, `X-Webhook-Event: <eventType>`, `X-Webhook-Signature: sha256=<hex hmac>` (HMAC-SHA256 of the raw JSON body using the endpoint's `secret` — this is what lets the receiver verify authenticity, same principle as verifying Twilio's own signature in Prompt 03).
4. Records a `WebhookDelivery` row per attempt.
5. On success: reset `consecutiveFailures` to 0, `lastStatus:'success'`.
6. On final failure (all attempts exhausted): increment `consecutiveFailures`, `lastStatus:'failure'`; if `consecutiveFailures` crosses a threshold (e.g. 20 consecutive full-failure event deliveries), auto-set `status:'disabled'` and fire an in-app `Notification` (Prompt 08) telling the owner their webhook was disabled — don't let a permanently-broken endpoint silently eat queue capacity forever.

## Event catalog (extend as needed to match whatever earlier prompts actually emit)

`call.completed`, `call.failed`, `campaign.completed`, `workflow.executed`, `lead.created`, `knowledge.document.ready`, `knowledge.document.failed`. Each payload should include enough denormalized context to be useful without a follow-up API call (e.g. `call.completed` includes duration, outcome, summary if available — not just an id).

## API — `/api/webhooks` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | list endpoints (secret masked, e.g. `whsec_****abcd`) |
| POST | `/` | create — returns the **full** secret once, in this response only |
| PATCH | `/:id` | update `url`/`events`/`status` |
| DELETE | `/:id` | delete |
| POST | `/:id/test` | fire a synthetic test event immediately, return the delivery result inline (bypass the queue for this one case so the UI can show pass/fail right away) |
| GET | `/:id/deliveries` | paginated delivery log for debugging |

## Environment variables
`REDIS_URL` (BullMQ, per Prompt 09).

## Acceptance checklist
- [ ] A registered endpoint receives a correctly-signed POST within seconds of the subscribed event occurring.
- [ ] The `X-Webhook-Signature` header verifies correctly against the returned secret using standard HMAC-SHA256.
- [ ] A consistently-failing endpoint auto-disables after the failure threshold and notifies the owner.
- [ ] `POST /:id/test` never touches real call/campaign data and returns a clear pass/fail immediately.
- [ ] A slow/hanging external endpoint never blocks or delays real call processing (verify the delivery is fully async).
