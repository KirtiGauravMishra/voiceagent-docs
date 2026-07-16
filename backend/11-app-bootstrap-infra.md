# 11 — App Bootstrap, Middleware Pipeline & Infrastructure

> Backend domain doc. Source: `backend/src/{app.ts, index.ts, config/{env,db,redis}.ts, middlewares/{error,timeout,requestId,httpLogger,rateLimiter}.middleware.ts}`.

> **Scope note**: `app.ts` also mounts the billing system's routers alongside the main-product routers — billing itself is out of scope for this documentation pass (see other files), but mentioned here where relevant to accurately describe the bootstrap sequence.

## 1. Overview

Single Express app (`createApp()` in `app.ts`), single Node process bootstrap (`index.ts`) that — depending on a `ROLE` env var — runs the HTTP server, the BullMQ background workers, or both. There is no separate framework-level "worker service" codebase; it's the same codebase running in a different mode.

## 2. Middleware Pipeline (`app.ts::createApp`)

Order matters here — each layer is applied in this exact sequence:

1. **`app.set('trust proxy', 1)`** — trusts one hop of `X-Forwarded-For` (needed for correct client-IP resolution behind Replit's/any reverse proxy — rate limiting and security-audit-log IP capture both depend on this being right).
2. **`requestId`** middleware — first, so every later log line and error response can carry an `X-Request-ID` (reads an incoming header if present — e.g. from a load balancer — else generates a `randomUUID()`; always echoed back as a response header too).


4. **Swagger UI** at `/api-docs` (+raw spec at `/api-docs.json`) — served **before** Helmet specifically so Helmet's CSP doesn't block Swagger's inline scripts/styles; a locally relaxed CSP header is set just for this one route.
5. **Helmet** — strict CSP (`default-src 'self'`, `connect-src` allow-listing the configured frontend origins + `*.vercel.app`), `crossOriginEmbedderPolicy: false` (needed for the widget/media-stream cross-origin use cases elsewhere in the app).
6. **CORS** — custom origin-callback (not a static list) — see §3.
7. **Compression** (`compression()`, threshold 1KB).
8. **Body parsers** — `express.json({limit:'1mb'})` + `urlencoded`. Note the 1MB cap: fine for normal API payloads, but any endpoint accepting large text blobs should use `multipart/form-data` (multer) rather than JSON, which several upload endpoints already do (knowledge docs, avatars, CSV imports).
9. **`httpLogger`** — custom access logger, skipped entirely in `NODE_ENV=test`.
10. **Global per-IP rate limiter** (`globalApiLimiter`) mounted on `/api` — see §4.
11. **`/health`** (liveness — no DB touch, instant `200 {status:'ok', pid}`) and **`/ready`** (readiness — pings Mongo `readyState`/`asPromise()` and Redis `ping()` in parallel, each with its own short timeout (2s/1s), returns `503` if either check fails) — the standard liveness/readiness split expected by most container orchestrators (Replit's health checks, if configured, should point at `/health` for "is the process alive" and optionally `/ready` for "should traffic be routed here").
12. **Route mounting** — all business routers under `/api/...` (main-product routers: `auth`, `agents`, `calls`, `widget`, `conversations`, `knowledge`, `knowledge-gaps`, `contacts`, `leads`, `phone-numbers`, `campaigns`, `voices`, `workflows`, `settings`, `notifications`, plus the public `contact`/`demo` routers and the Twilio webhook router — see each domain's own doc for full endpoint detail; `billing` and a Brevo `webhooks` router are mounted here too but are out of scope for this documentation pass).
13. **`serveAudioFiles(app)`** — serves pre-generated TTS mp3s at `/audio/:file` (see `03-calls-realtime-pipeline.md`).
14. **`/uploads`** static file serving (`express.static`) — local-disk-served user uploads; confirm this still makes sense once/if any Cloudinary→S3 storage migration happens (a static-served local uploads folder is a separate concern from the Cloudinary-hosted avatar/receipt assets documented elsewhere).
15. **`notFoundHandler`** then **`errorHandler`** — always last, in that order.

## 3. CORS policy (`app.ts`, lines ~53-96)

A custom origin-callback function, evaluated in order:
1. **No `Origin` header** → allowed (server-to-server calls, Postman/curl, and — importantly — this is also what lets the Twilio webhook requests through, since Twilio doesn't send an Origin header).
2. **Exact match** against the comma-separated `FRONTEND_URL` env var.
3. **Any `*.vercel.app` subdomain** — a blanket allow for Vercel preview deployments (reasoned as "safe" since Vercel preview subdomains are unique per-deployment) — **only relevant if the rebuild's frontend still deploys to Vercel**; drop this rule if the in-house rebuild serves its frontend from Replit or elsewhere instead.
4. **Known localhost dev ports** (`5173`,`4173`,`3000`) — only outside `NODE_ENV=production`.
5. Anything else → rejected with a thrown `Error` (surfaces as a CORS failure in the browser, not a clean JSON 403 — standard `cors` package behavior).

Allowed headers include `X-Request-ID` (so the frontend could echo/read it, though nothing in the frontend docs currently does); exposed headers include the rate-limit headers (`X-RateLimit-*` — though note the actual in-memory rate limiter, §4, does **not** currently set these specific header names — it sets `Retry-After` on a 429. This CORS `exposedHeaders` entry appears to be aspirational/leftover rather than reflecting current limiter behavior; worth reconciling one way or the other during rebuild.)

## 4. Rate Limiting (`middlewares/rateLimiter.middleware.ts`)

A **hand-rolled in-memory token-bucket** limiter — no Redis, no external dependency, "zero latency overhead on every request" per the source comment. A single `Map<string, {tokens, last}>` keyed by `${prefix}:${clientIp}`, refilled continuously based on elapsed time × `max/windowMs` rate, with entries older than 1 hour swept every 5 minutes.

| Limiter | Window | Max | Applied to |
|---|---|---|---|
| `globalApiLimiter` | 60s | 300 | every `/api/*` request |
| `loginRateLimiter` | 15min | 25 | `/auth/login`, `/auth/login/2fa`, `/auth/verify-email`, `/auth/forgot-password` |
| `registerRateLimiter` | 1hr | 5 | `/auth/register` |
| `refreshRateLimiter` | 15min | 30 | `/auth/refresh` |
| `callDispatchLimiter` | 60s | 20 | `POST /api/calls` |

**Explicitly documented tradeoff**: this is per-process, in-memory state. On a single Replit instance this is exactly as strong as intended; if the rebuild ever runs multiple concurrent instances behind a load balancer without sticky sessions, each instance enforces its own independent bucket — a client could get up to `N×` the intended limit where `N` is the instance count. The original codebase's own comment accepted this tradeoff at 2 instances; re-evaluate if the scale requirements below push toward a larger instance count and these limits start to matter for abuse-prevention rather than just UX.

## 5. Error Handling (`middlewares/error.middleware.ts`)

Two special-cased response shapes:
- **Twilio webhook paths** (`/api/twilio/{inbound,voice,gather,status,recording}*`) — both the 404 handler and the error handler detect these paths and always respond `200` with fallback TwiML (`<Say>...</Say><Hangup/>`), **never** a JSON error — a non-200/non-TwiML response makes Twilio play its own generic "application error" tone to a live caller, which this design avoids even for genuinely unexpected server errors.
- **Everything else**: `AppError` subclasses (see `01-auth.md §7`) map to their `statusCode`/`code`/`message`/`details`; anything else (an unhandled exception) becomes a generic `500 INTERNAL_ERROR` with no internal detail leaked to the client. 5xx-class errors are logged at `error` level (with the full error object and request ID); 4xx-class `AppError`s are logged at `warn` with just the code/path (expected-error noise reduction).

## 6. Process Bootstrap (`index.ts`)

### `ROLE` env var
```
ROLE=api     → HTTP server only, no BullMQ workers
ROLE=worker  → BullMQ workers only, minimal health-only HTTP server (still binds a port)
ROLE=all     (default, unset) → both, in one process
```
For a single Replit deployment, `ROLE=all` (the default) is almost certainly the right choice — running separate `api`/`worker` processes only makes sense once you're horizontally scaling the two concerns independently (e.g. many API instances behind a load balancer, fewer/dedicated worker instances), which is a later-stage optimization, not a starting point.

### Boot sequence
1. `connectRedis()` + `connectDB()` **in parallel** (cuts cold-start latency vs. sequential connects).
2. **Seeding** (only on `runApi`, to avoid two processes racing on an empty collection simultaneously): platform seed scripts run here — some of what's seeded (plans, bank accounts) is billing-specific and out of scope; keep whichever seed scripts are still relevant to the in-house rebuild and drop the rest.
3. **Stuck-call cleanup** (only on `runApi`): runs once immediately, then every 2 minutes. Finds `Call`s stuck `in-progress` for >15 minutes (a call that never got a terminal status callback — e.g. Twilio's webhook was lost, or the process crashed mid-call), estimates a safe duration (stored value → transcript-timestamp-based estimate + 30s buffer → agent's `maxDurationSeconds` as the final fallback), marks them `completed`, and re-runs post-call billing/processing. This is a real correctness safety-net for the call pipeline (documented in more depth in `03-calls-realtime-pipeline.md §10`) — **keep this** in the rebuild regardless of what happens to billing (the billing call within it should be stubbed/removed per that doc's rebuild notes, but the underlying "don't leave calls stuck forever" cleanup logic is still needed).
4. `ensureAtlasVectorIndex()` (only on `runApi`) — fire-and-forget KB vector index provisioning, see `07-knowledge-base.md`.
5. **BullMQ workers** started (only on `runWorker`): call, campaign, workflow-followup, workflow-scheduler (+ a lead-sequence worker not covered by this documentation pass — drop if that system is fully removed).
6. **Workflow-followup Redis-sorted-set poller** and **workflow-scheduler 60s tick** — started on `runApi` **or** `runWorker` (i.e. whichever process(es) are running, at least one of them runs these) — see `06-workflows.md §5,7`.
7. **Log pruning** (hourly) and a **subscription-automation email sweep** (every 6h, billing-specific — drop if billing is fully removed) — both `runApi`-only.
8. **HTTP server**: if `runApi`, creates a raw `http.Server` wrapping the Express app (`keepAliveTimeout:65s`, `headersTimeout:66s` — deliberately set higher than the default 5s/6s to avoid premature connection drops on slow client networks, common advice when running behind certain load balancers/proxies), calls `attachMediaStreamServer(server)` to wire the WebSocket upgrade handler onto the **same** server/port (see `03-calls-realtime-pipeline.md §7`), then listens on `env.PORT`. If **not** `runApi` (pure worker role), starts a minimal `/health`-only HTTP server instead — some hosting platforms (the original codebase's Render deployment) require every deployed service to bind a port even if it's "just a worker"; confirm whether Replit's deployment model has the same requirement before assuming this stub server is still necessary.
9. **Graceful shutdown** (`SIGTERM`/`SIGINT`): clears all `setInterval`s, closes all BullMQ workers, disconnects Mongo + Redis, then exits — with a **30-second forced-exit timeout** as a backstop if graceful cleanup hangs.
10. **`unhandledRejection`** is logged but does not crash the process; **`uncaughtException`** logs at `fatal` and **does** exit the process (relying on the platform/process manager to restart it) — a deliberate "fail fast on truly unexpected state" policy for exceptions, vs. a more tolerant policy for rejected promises.

## 7. Environment Variables (`config/env.ts`)

Validated with **Zod** at boot — `process.exit(1)` immediately if validation fails, with the specific field errors printed to console. Loads `.env` from both the current working directory and one level up (supports running `npm run dev` from either the repo root or `backend/`).

Notable parsing details:
- `stripToRedisUri()` — defensively strips dashboard-paste artifacts (`redis-cli -u `, wrapping quotes, URL-encoded junk) from `REDIS_URL` before ioredis ever sees it. Worth keeping this exact defensive parsing in a rebuild — it's cheap insurance against a very common copy-paste mistake when moving Redis connection strings between environments/providers.
- `trimQuotes()` — strips accidental wrapping quotes from several other string env vars (a dashboard-paste artifact, not Redis-specific).

### Full env var reference (main-product-relevant subset; see individual domain docs for the "why" behind each)
| Variable | Domain doc |
|---|---|
| `NODE_ENV`, `PORT` | this file |
| `MONGODB_URI`, `TEST_MONGODB_URI` | this file / `01-auth.md` |
| `REDIS_URL` | `09-jobs-queues.md` |
| `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `JWT_ACCESS_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | `01-auth.md` |
| `FRONTEND_URL`, `PUBLIC_APP_URL` | this file (§3) |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `DEEPGRAM_API_KEY`, `CARTESIA_API_KEY`, `ELEVENLABS_API_KEY` | `03-calls-realtime-pipeline.md` |
| `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_API_KEY`, `TWILIO_API_SECRET`, `TWILIO_PHONE_NUMBER`, `TWILIO_TWIML_APP_SID`, `TWILIO_WEBHOOK_BASE_URL` | `03-calls-realtime-pipeline.md` |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` | `03-calls-realtime-pipeline.md` (Polly), rebuild note below (SES) |
| `BREVO_*` | `01-auth.md` (rebuild: swap for SES) |
| `CLOUDINARY_*` | `01-auth.md`/`02-agents.md`/`08-notifications-settings.md` (rebuild: swap for S3) |
| `RAZORPAY_*`, `PAYPAL_*`, `DODO_*` | billing — out of scope |

## 8. MongoDB connection tuning (`config/db.ts`)

`maxPoolSize:100, minPoolSize:5, maxIdleTimeMS:60s, serverSelectionTimeoutMS:5s, socketTimeoutMS:45s, connectTimeoutMS:5s, heartbeatFrequencyMS:10s, family:4 (force IPv4), waitQueueTimeoutMS:5s, bufferCommands:true`. A ping against the admin DB is measured and logged immediately after connecting — a lightweight diagnostic for catching a bad region choice (e.g. app deployed far from the Mongo cluster) at boot rather than discovering it from slow request latency later.

## 9. Scale Requirement: 100,000+ calls/day, 1,000+ concurrent calls

This is a stated hard target for the in-house rebuild (raised from an earlier 25k/day, 400+ concurrent target; see also `03-calls-realtime-pipeline.md §13` for the call-pipeline-specific implications). From an infrastructure/bootstrap perspective specifically:

- **MongoDB pool sizing**: the current `maxPoolSize:100` was tuned for a much smaller SaaS workload and is almost certainly far too low at this scale. At 1,000+ concurrent calls, each writing transcript entries multiple times per minute (plus all the normal API traffic on top), plan for a materially larger pool (likely spread across multiple app instances rather than one giant pool on one process) and load-test against measured connection usage rather than guessing a number — at this volume, also evaluate whether a sharded/clustered MongoDB topology (vs. a single replica set) is needed.
- **Single-instance vs. multi-instance**: at 1,000+ concurrent calls, `ROLE=all` on a single instance is very unlikely to be sufficient regardless of instance size — the real-time media-stream pipeline's CPU/network profile (concurrent TTS synthesis, WebSocket fan-out) becomes the binding constraint well before 1,000 concurrent calls on any single reasonably-priced instance (see `03-calls-realtime-pipeline.md §13`). Plan for `ROLE=api`/`ROLE=worker` split processes from the outset (already supported by the existing `ROLE` env var design — no code change needed for that split itself), and for multiple `ROLE=api` instances behind a load balancer, which requires solving the in-memory-state issues flagged in `03-calls-realtime-pipeline.md §13` and the rate-limiter caveat in §4 above as a first-class part of the rebuild, not a later optimization.
- **Rate limits**: `globalApiLimiter` (300 req/60s per IP) and `callDispatchLimiter` (20 dispatches/60s per IP) were sized for a multi-tenant SaaS where each customer is a different IP. For an in-house tool, reconsider whether IP-based limiting is even the right model (e.g. if all calls are dispatched from one internal server-to-server integration behind a single IP, the current `callDispatchLimiter` would throttle *all* internal traffic collectively at 20/min — dramatically too low for a 100k-calls/day target, which averages ~69/min sustained and will have much sharper burst peaks at 1,000+ concurrent). This limiter's threshold needs a full rethink against the real traffic pattern the rebuild will actually see — the current SaaS-tuned defaults don't just need tuning, they need redesigning (e.g. a per-integration or global-service-account limit rather than per-IP, sized in the hundreds/thousands per minute, not tens).
- **Health checks**: keep `/health` (liveness) cheap and DB-independent as it is today — under load, you want liveness checks to keep passing even if the DB is briefly slow, so an orchestrator doesn't kill-and-restart a healthy-but-busy instance. At this concurrency, also make sure the orchestrator's health-check interval/timeout is tuned so a momentarily-busy (not dead) instance under heavy call load isn't mistakenly cycled.

## 10. Rebuild Notes (Replit target, in-house/no-billing)

### Dependencies
`express`, `helmet`, `cors`, `compression`, `mongoose`, `ioredis`, `zod`, `dotenv`, `swagger-ui-express`/`swagger-jsdoc` (dev-docs only, safe to keep or drop independent of everything else here).

### Deployment target change: Render → Replit
The codebase currently carries several Render-specific artifacts/assumptions (a health-only stub server for "worker" role because "Render requires a listening port even for Web Service workers", the in-memory rate-limiter's "2 Render instances" framing, a `render.yaml` at the repo root). None of these need to change functionally for Replit, but confirm Replit's own deployment model before assuming every one of these workarounds is still necessary — e.g. verify whether a Replit "worker" deployment (if such a distinction exists in whatever Replit deployment product is used) also requires a bound port, or whether that stub server can be simplified away.

### Storage/email provider swaps
Already covered per-domain (`01-auth.md`, `02-agents.md`, `08-notifications-settings.md`): Brevo → AWS SES for transactional email, Cloudinary → S3 for file storage. This file's env-var table (§7) is the consolidated map of which vars change as part of that swap.

### What to drop entirely if billing is fully removed
- Route mounts: `billingRoutes`, `webhookRoutes` (Brevo) in `app.ts`.
- The path-specific long timeouts in the timeout middleware (all target bulk-import routes not covered by this documentation pass).
- Seed scripts: plans, bank accounts — keep `seedSettings` only if `PlatformSettings` is still used by something in-scope (confirm before dropping — it's also read by the public contact-form endpoint documented in `05-leads-contacts-crm.md`).
- The subscription-automation email sweep interval.
- The `leadSequence` worker.
- `RAZORPAY_*`/`PAYPAL_*`/`DODO_*`/`GOOGLE_PLACES_API_KEY` env vars.
