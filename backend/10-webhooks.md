# 10 — Webhooks (Inbound, from external providers)

> Backend domain doc. Source: `backend/src/{controllers/{twilio,webhook}.controller.ts, routes/{twilio,webhook}.routes.ts}`. This is a consolidated index/security-review of every "external service calls us" endpoint in the main product. Full behavioral detail for the Twilio webhooks (by far the most important ones) lives in `03-calls-realtime-pipeline.md §4-10` — this file adds the security/conventions view across all of them and doesn't repeat that detail.

## 1. Overview

A "webhook" here means: an endpoint mounted **without** the `authenticate` JWT middleware, reachable by an external service pushing an event at us. Two provider integrations expose webhooks in the current codebase:

| Provider | Mount | In scope now? |
|---|---|---|
| **Twilio** | `/api/twilio/*` | **Yes** — core to the product (call flow) |
| **Brevo** (transactional email) | `/api/webhooks/brevo` | No — feeds a different, currently out-of-scope panel; not covered here |

(Payment-provider webhooks — Razorpay/PayPal/Dodo, mounted under `/api/billing/webhook/*` — exist in the codebase but are entirely part of the billing/subscription system, which is out of scope for this documentation pass. Skip them for now; revisit alongside billing if that domain comes back into scope.)

## 2. Twilio webhooks — full reference table

All mounted at `/api/twilio` (`twilio.routes.ts`), **all public** (no `authenticate` middleware — Twilio has no way to send a JWT). Full behavior for each is in `03-calls-realtime-pipeline.md`.

| Method | Path | Fires when | Doc reference |
|---|---|---|---|
| GET/POST | `/voice/:callId` | Call connects, Twilio requests what to do | §5 |
| GET | `/greeting-audio/:callId` | Twilio's `<Play>` fetches pre-generated greeting audio | §5 |
| POST | `/gather/:callId` | User's speech was transcribed (non-streaming fallback mode) | §8 |
| POST | `/status/:callId` | Any call status transition | §10 |
| POST | `/recording/:callId` | A call recording finished processing | §10 |
| POST | `/inbound/:phoneNumberId` | Inbound call to an owned number | §9 |
| POST | `/inbound` | Inbound call to an unrecognized/generic number | §9 |
| POST | `/widget/voice-twiml` | Browser-widget (Twilio Voice SDK) call connects | §9 |
| GET | `/recording-proxy` | Frontend fetches a recording's audio (proxied, not a provider-initiated webhook, but public for the same reason) | §7 |

### Security model — **no signature validation currently implemented**
`twilio.controller.ts`'s file header comment states outright: *"In production, validate Twilio request signature using X-Twilio-Signature"* — this is written as a to-do, not as documentation of something already done. **As of this read, none of the Twilio routes validate the `X-Twilio-Signature` header.** Any of these endpoints can currently be called by anyone who guesses/enumerates a valid `callId` or `phoneNumberId` — there's no cryptographic proof a request actually originated from Twilio.

Practical exposure, endpoint by endpoint:
- `/voice`, `/gather`, `/greeting-audio` — a forged call would only affect a `Call` document that's already fake/attacker-controlled state; limited blast radius, but could be used to probe system prompts/behavior.
- `/status`, `/recording` — a forged request could **mark a call completed and trigger billing/notifications/workflow-triggers for a call that never happened**, or overwrite a real call's recording URL with an attacker-supplied one — the higher-value target for signature validation if only one endpoint gets hardened first.
- `/inbound/:phoneNumberId` — a forged request could **place a fake "inbound call"** against a real `PhoneNumber`, consuming quota and creating a spurious `Call`+`Contact`.
- `/recording-proxy` — proxies **any** URL passed via `?url=`, not just Twilio's own recording URLs, with the platform's Twilio Basic-Auth credentials attached server-side. This is the most concerning of the group: it's an authenticated-fetch proxy with no allow-list on the target host beyond "starts with something a URL parser accepts" — worth an explicit look during a security pass (constrain `url` to Twilio's own recording-domain pattern, e.g. `*.twilio.com`, before forwarding credentials).

### The "always return 200 + valid TwiML" convention
Every voice-response endpoint (`voice`, `gather`, `inbound*`, `widget/voice-twiml`) returns **HTTP 200 with a fallback TwiML `<Say>`+`<Hangup>` on any internal error** rather than a 4xx/5xx — documented in `03-calls-realtime-pipeline.md §4`, repeated here because it's directly relevant to the security posture: **application-level errors are deliberately invisible at the HTTP layer** on these routes. This is correct behavior for Twilio's own retry/error semantics, but means a monitoring setup watching for 5xx rates on `/api/twilio/*` will see nothing even when things are actively failing — failures must be observed via **application logs** (`logger.error` calls at each catch site) instead of HTTP status codes.

### Rate limiting
Twilio routes are **not** behind any route-specific rate limiter (unlike `/api/auth/*` or `/api/calls`) — only the global `/api` per-IP limiter (`app.ts`) applies, and Twilio's own IPs will be shared across every customer's calls, so a per-IP limit doesn't meaningfully protect any single endpoint here from abuse via a guessed `callId`.

## 3. Brevo webhook (out of scope)

`POST /api/webhooks/brevo` exists in the codebase (email open/click tracking) but its only consumer is a currently out-of-scope panel — not detailed here.

## 4. Rebuild Notes (Replit target, in-house/no-billing)

### Priority hardening item: Twilio signature validation
If this rebuild will run with real Twilio credentials handling real calls, implementing `X-Twilio-Signature` validation (Twilio's official `twilio.validateRequest()` helper, using the account's Auth Token) on at minimum `/status` and `/recording` (the two endpoints that trigger billing/state changes) should be treated as a pre-launch security item, not deferred indefinitely — the current codebase ships without it.

### `recording-proxy` allow-list
Constrain the `?url=` parameter to Twilio's own recording URL shape before forwarding the platform's Basic-Auth Twilio credentials to it — currently any URL is accepted and proxied with those credentials attached.

### Billing webhooks
Once/if billing is back in scope, `POST /api/billing/webhook/{razorpay,paypal,dodo}` will need their own doc — they **do** implement signature verification today (HMAC for Razorpay, PayPal's server-side verify-signature API, a webhook key for Dodo), unlike the Twilio routes, so that part doesn't need rebuild attention — only the routes/logic themselves would need documenting if billing returns to scope.

### Dependencies
`twilio` SDK (for `validateRequest()`, if added), no new dependencies otherwise — Brevo webhook uses only `express`/`mongoose`, already present.
