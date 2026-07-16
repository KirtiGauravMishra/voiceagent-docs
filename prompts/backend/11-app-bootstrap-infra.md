# Backend Prompt 11 — App Bootstrap, Middleware Pipeline & Infrastructure

> Read `../00-master-rebuild-guide.md` and all prior backend prompts first — this one wires them all together into one running process.

## Goal
One Express app + one Node process bootstrap that can run the HTTP API, the BullMQ background workers, or both, deployed as a single Replit deployment.

## Middleware pipeline (`createApp()`) — build in this exact order

1. `app.set('trust proxy', 1)` — needed for correct client-IP resolution behind Replit's reverse proxy (rate limiting and audit-log IP capture depend on this).
2. **Request-ID middleware** — first, so every later log line and error response carries an `X-Request-ID` (read an incoming header if present, else generate one; echo it back as a response header).
3. Optional API-docs route (Swagger UI or equivalent) if you want one — mount it **before** Helmet with a relaxed CSP scoped to just that route, since Helmet's default CSP blocks inline scripts/styles docs UIs typically need.
4. **Helmet** — strict CSP (`default-src 'self'`, `connect-src` allow-listing your actual frontend origin(s)), `crossOriginEmbedderPolicy: false` (needed for the media-stream WebSocket use case).
5. **CORS** — origin-callback allowing: no-`Origin` requests (server-to-server calls, and required for the Twilio webhook requests, which never send an `Origin` header), your configured `FRONTEND_URL`(s) exactly, and localhost dev ports outside production. Reject everything else.
6. **Compression** (threshold ~1KB).
7. **Body parsers** — `express.json({limit:'1mb'})` + `urlencoded`. Anything larger than 1MB (file uploads, CSV imports) goes through multipart/multer, not JSON.
8. **HTTP access logger**, skipped in test env.
9. **Global per-IP rate limiter** on `/api/*` (see limits table below).
10. `/health` (liveness — no DB touch, instant `200`) and `/ready` (readiness — pings Mongo + Redis in parallel with short timeouts, `503` if either fails). Point your Replit health check at `/health`.
11. **Route mounting** — all routers under `/api/...`: `auth`, `agents`, `calls` (+ the public Twilio webhook router), `contacts`, `leads`, `campaigns`, `phone-numbers`, `voices`, `workflows`, `knowledge`, `settings`, `notifications`, `webhooks`.
12. Static audio file serving for pre-generated TTS mp3s (Prompt 03).
13. `notFoundHandler` then `errorHandler` — always last, in that order.

## Rate limiting

Build a Redis-backed limiter (not in-memory — a single-process in-memory limiter breaks the moment you run more than one instance, which this scale target requires from day one).

| Limiter | Window | Suggested max | Applied to |
|---|---|---|---|
| `globalApiLimiter` | 60s | a few thousand per key | every `/api/*` request |
| `loginRateLimiter` | 15min | ~25 | login/2FA/verify-email/forgot-password |
| `registerRateLimiter` | 1hr | ~5 | register |
| `refreshRateLimiter` | 15min | ~30 | token refresh |
| `callDispatchLimiter` | 60s | sized in the hundreds, not tens | `POST /api/calls` |

Size `callDispatchLimiter` against the real target: 100k calls/day averages ~69/min sustained with much sharper bursts — a limit in the tens will throttle legitimate traffic. If most dispatch traffic comes from one internal integration rather than many distinct external IPs, key the limiter by API key/service-account instead of raw IP.

## Error handling

- **Twilio webhook paths** (`/api/twilio/*`) — both the 404 handler and the error handler must respond `200` with fallback TwiML (`<Say>...</Say><Hangup/>`) on any error, **never** a JSON error — a non-TwiML response makes Twilio play a generic error tone to a live caller.
- **Everything else** — `AppError` subclasses map to their `statusCode`/`code`/`message`/`details`; unhandled exceptions become a generic `500 INTERNAL_ERROR` with no internal detail leaked. Log 5xx at `error` level with full context; log 4xx `AppError`s at `warn` with just code/path.

## Process bootstrap (`index.ts`)

### `ROLE` env var
```
ROLE=api     → HTTP server only, no BullMQ workers
ROLE=worker  → BullMQ workers only, minimal health-only HTTP server
ROLE=all     (default) → both, in one process
```
Start with `ROLE=all` for initial development; plan to split into separate `api`/`worker` deployments once load-testing shows the media-stream pipeline's CPU/network profile needs isolation from the API tier (see the scale section below — at 1,000+ concurrent calls this split stops being optional).

### Boot sequence
1. Connect Redis and Mongo **in parallel**.
2. Any one-time seed data (only from the `api` role, to avoid two processes racing on an empty collection).
3. **Stuck-call cleanup**: run once immediately, then every 2 minutes (only from the `api` role). Find `Call`s stuck `in-progress` for >15 minutes, estimate a safe final duration (stored value → transcript-timestamp estimate + 30s buffer → agent's `maxDurationSeconds` as last resort), mark them `completed`, run post-call processing. This is a real correctness safety net for the call pipeline (Prompt 03) — don't skip it.
4. Fire-and-forget knowledge-base vector index provisioning (Prompt 07), `api` role only.
5. Start BullMQ workers (`worker` role only) — call, campaign, workflow-followup, knowledge-ingest, webhook-delivery, post-call-processing (Prompt 09).
6. Start the workflow-followup Redis-sorted-set poller and the workflow-scheduled-tick (Prompt 06) — from whichever role(s) are actually running, at least one instance must run these.
7. Hourly log pruning (`api` role only), if you're persisting structured logs to a DB collection.
8. **HTTP server**: if running the `api` role, create a raw `http.Server` wrapping the Express app with `keepAliveTimeout:65s`/`headersTimeout:66s` (higher than Node's defaults, to avoid premature connection drops behind a proxy), attach the media-stream WebSocket upgrade handler to the same server (Prompt 03 §7), then listen on `PORT`. If running worker-only, bind a minimal `/health`-only HTTP server instead (confirm whether your Replit deployment type actually requires this before assuming it's necessary).
9. **Graceful shutdown** on `SIGTERM`/`SIGINT`: clear all intervals, close all BullMQ workers, disconnect Mongo + Redis, then exit — with a ~30s forced-exit timeout as a backstop.
10. Log (don't crash on) `unhandledRejection`; log at `fatal` and **do** exit on `uncaughtException` (let the platform restart the process) — fail fast on truly unexpected state, stay tolerant of rejected promises.

## Environment variable validation

Validate all env vars at boot with Zod (or similar) — `process.exit(1)` immediately with the specific field errors printed if validation fails, rather than limping along with `undefined` config values. Defensively strip common copy-paste artifacts from connection strings (wrapping quotes, CLI-flag prefixes like `redis-cli -u `, URL-encoded junk) before handing them to the Redis/Mongo clients — a cheap guard against a very common mistake when moving connection strings between environments.

### Consolidated environment variable reference
| Variable | Covered in |
|---|---|
| `NODE_ENV`, `PORT` | this file |
| `MONGODB_URI` | this file / `01-auth.md` |
| `REDIS_URL` | `09-jobs-queues.md` |
| `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `JWT_ACCESS_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | `01-auth.md` |
| `FRONTEND_URL` | this file |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `DEEPGRAM_API_KEY`, `CARTESIA_API_KEY`, `ELEVENLABS_API_KEY` | `03-calls-realtime-pipeline.md` |
| `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_API_KEY`, `TWILIO_API_SECRET`, `TWILIO_PHONE_NUMBER`, `TWILIO_TWIML_APP_SID`, `TWILIO_WEBHOOK_BASE_URL` | `03-calls-realtime-pipeline.md` |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET`, `SES_FROM_EMAIL` | master guide (SES + S3) |

## MongoDB connection tuning

Set `maxPoolSize`, `minPoolSize`, `maxIdleTimeMS`, `serverSelectionTimeoutMS`, `socketTimeoutMS`, `connectTimeoutMS`, `heartbeatFrequencyMS`, `family:4` (force IPv4), `waitQueueTimeoutMS`, `bufferCommands:true` explicitly rather than relying on driver defaults. Log a boot-time ping latency against the admin DB — a cheap way to catch a bad region choice (app deployed far from the Mongo cluster) before it shows up as mysterious request latency later.

## Scale requirement: 100,000+ calls/day, 1,000+ concurrent calls

Build this multi-instance-ready from the start, not as a later optimization:
- **Mongo pool sizing**: size the connection pool against measured usage under load test, spread across multiple app instances rather than one giant pool on one process; evaluate a sharded/clustered topology, not just a single replica set, once you're validating at this volume.
- **Single vs. multi-instance**: `ROLE=all` on one instance will not sustain 1,000+ concurrent calls — the real-time media pipeline's CPU/network profile (concurrent TTS synthesis, WebSocket fan-out) becomes the binding constraint well before that. Plan for separate `ROLE=api`/`ROLE=worker` processes and multiple `api` instances behind a load balancer, which requires the Redis-backed rate limiting and shared-state discipline already specified above and in Prompt 03.
- **Rate limits**: don't carry over limiter numbers sized for a small multi-tenant SaaS — redesign `callDispatchLimiter` specifically around this system's actual traffic pattern (see the rate-limiting section above).
- **Health checks**: keep `/health` cheap and DB-independent so a busy-but-alive instance under heavy call load is never mistakenly killed by the orchestrator; tune the orchestrator's check interval/timeout accordingly.

## Environment variables
See the consolidated table above.

## Acceptance checklist
- [ ] `ROLE=api`, `ROLE=worker`, and `ROLE=all` each boot correctly and do only the work their role implies.
- [ ] A Twilio webhook request never triggers a CORS rejection or a JSON error response, even on an internal failure.
- [ ] Killing the process with `SIGTERM` completes in-flight jobs and shuts down cleanly within the forced-exit timeout.
- [ ] A stuck `in-progress` call older than 15 minutes is auto-completed by the sweep within one cycle.
- [ ] Missing or malformed required env vars cause an immediate, clearly-messaged boot failure rather than a runtime crash later.
- [ ] `/health` stays fast and passing even when Mongo/Redis are briefly slow; `/ready` correctly reports `503` when either is down.
