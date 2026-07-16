# 00 — Master Rebuild Guide (read this before any section prompt)

> This is the orchestration prompt for rebuilding **VoiceAgent**, an AI voice-calling platform, as an **in-house tool on Replit**. Every other file in this `prompts/` folder builds one section (one backend domain or one frontend area) and assumes you have already read and applied everything in this file. If you are an AI coding agent picking up a single section prompt in isolation, read this file first — it defines the shared conventions every section must follow so the sections fit together into one coherent app.

## 1. What you're building

An internal AI voice-agent platform: create "agents" (LLM + TTS voice personas), place/receive phone calls through them in real time (speech-to-text → LLM → text-to-speech, streamed over a WebSocket for low latency), run outbound calling campaigns against contact lists, track leads through a sales pipeline, build simple call-triggered automations ("workflow" chains), maintain a knowledge base agents can search mid-call (RAG), and monitor everything through a dashboard.

This is **not** a multi-tenant SaaS product. It is a single-organization internal tool. Design accordingly:
- No billing, no subscription plans, no payment providers, no per-feature plan gating.
- No superadmin portal, no employee/sales-CRM portal. One user role is enough (`user`). Do not build multi-role auth, impersonation, or role-based portals.
- No public embeddable third-party widget (the chat/voice widget that gets dropped into someone else's website). If a browser-based calling UI is wanted, it's part of the main authenticated app, not a public unauthenticated embed.

## 2. Hard scale requirement — design for this from day one

**The platform must sustain 100,000+ calls/day with 1,000+ of them concurrent at peak.** This is not a "nice to have later" — it changes fundamental architecture decisions you must make while building, not retrofit afterward:

- **Never store live-call session state in a plain in-process `Map`.** The real-time call pipeline's per-call state (conversation history, streaming/barge-in control flags, active WebSocket handles) must be designed to work across multiple horizontally-scaled instances from the start — back it with Redis (or accept sticky-routing-per-call at the load balancer as a documented constraint, but still keep state externally recoverable). Building it in-memory "for now" at this scale is not a shortcut, it's a rewrite waiting to happen.
- **Never write generated files (TTS audio, uploads) to local instance disk and assume the same instance serves them later.** Use S3 (see §5) or regenerate-on-demand.
- **Any per-IP or per-process rate limiter must be sized for hundreds–thousands of requests/minute, not tens.** Think in terms of internal service-to-service call volume, not "one customer's browser."
- **MongoDB connection pooling and index design must assume heavy sustained write throughput** (every call turn writes a transcript entry; at 1,000+ concurrent calls this is constant). Plan for a properly sized pool and, if using self-hosted Mongo, a real replica set / sharding conversation — don't assume a default pool size is enough.
- **Assume multi-instance deployment for the API/worker processes from the outset.** A single instance is very unlikely to hold 1,000+ concurrent real-time call sessions (each needs a live WebSocket to the telephony provider + a live WebSocket to the speech-to-text provider, plus concurrent LLM/TTS calls) regardless of how large that one instance is — the bottleneck is CPU/network throughput for concurrent audio streaming, not just memory.
- External providers (telephony, speech-to-text, TTS) have their own concurrency ceilings independent of your code — flag in your implementation notes wherever a provider's plan tier needs to support 1,000+ concurrent connections, so this gets budgeted for separately from the application build.

## 3. Tech stack

**Backend**: Node.js + Express + TypeScript, MongoDB (Mongoose ODM), Redis, BullMQ for background job queues, `ws` for WebSocket handling, Zod or Yup for request validation, `bcryptjs` + `jsonwebtoken` for auth, `@otplib/preset-default` for TOTP 2FA.

**Frontend**: React 19 + TypeScript + Vite, Tailwind CSS, React Router v6, TanStack React Query, React Hook Form + Yup, Axios, Recharts for charts, `lucide-react` for icons.

**Real-time voice pipeline**: Twilio (telephony — calls, phone numbers, Media Streams), Deepgram (streaming speech-to-text), OpenAI + Anthropic (LLM), OpenAI TTS + Cartesia + ElevenLabs (text-to-speech, in that fallback order).

**Infrastructure**: Deployed on Replit. MongoDB Atlas recommended (enables native vector search for the knowledge base — see the knowledge-base section prompt). Redis for job queues, caching, and cross-instance call-session state (per §2).

## 4. Shared backend conventions (apply to every backend section)

### API response envelope
Every successful response: `{ "success": true, "data": <payload> }`. Paginated list responses: `{ "success": true, "data": [...], "pagination": { "page": N, "limit": N, "total": N, "totalPages": N } }`. Every error response: `{ "success": false, "error": { "code": "SOME_CODE", "message": "human message", "details": <optional> } }`.

### Error handling
Define a base `AppError` class (`message, statusCode, code, details`) with subclasses: `NotFoundError` (404, `NOT_FOUND`), `UnauthorizedError` (401, `UNAUTHORIZED`), `ForbiddenError` (403, `FORBIDDEN`), `ConflictError` (409, `CONFLICT`), `ValidationError` (422, `VALIDATION_ERROR`). A single global Express error-handling middleware (mounted last) catches these and formats the envelope above; anything else becomes a generic `500 INTERNAL_ERROR` with no internal detail leaked to the client. Log 5xx at `error` level with full detail; log 4xx `AppError`s at `warn` with just code/path.

### Auth middleware pattern
JWT access + refresh token pair (see the Auth section prompt for exact shape). An `authenticate` Express middleware reads `Authorization: Bearer <token>`, verifies it, and populates `req.user = {id, email, name}`. Every route that isn't explicitly public requires this middleware. Since there's only one role (`user`), no `requireRole` layer is needed — every authenticated resource is scoped by `ownerId: req.user.id` (or equivalent) at the query level, never by role.

### Mongoose model conventions
- Every model gets a `toJSON` transform that renames `_id` → `id` and strips `__v`, so the API never leaks Mongo's raw `_id`/`__v` naming to clients.
- Every owned resource (agents, calls, contacts, leads, campaigns, workflows, knowledge documents, etc.) has an `ownerId: ObjectId` field, and every read/write query filters by it. There is only one tenant, but keep the field — it's how you scope data per authenticated user session and it costs nothing to keep the pattern consistent.
- Sensitive fields (password hashes, OTP codes, 2FA secrets) are declared `select: false` so they're never accidentally returned by a normal `.find()`.
- Add compound indexes matching your actual query patterns (e.g. `{ownerId, createdAt: -1}` for "list my most recent X").

### Validation
Validate request bodies at the route layer with Yup or Zod schemas before the controller/service runs. Return `422 VALIDATION_ERROR` with field-level details on failure.

### Logging
Structured JSON logging (a simple pino-style logger is fine) with a request-ID stamped on every request (generate one if the client doesn't send `X-Request-ID`, echo it back as a response header, include it in every log line for that request).

### Background jobs
Use BullMQ for anything that shouldn't block an HTTP response: campaign dialing loops, scheduled workflow follow-ups, log pruning. Give every queue/worker pair a clear job-data TypeScript interface. Prefer a short-lived Redis connection per enqueue call over one shared long-lived connection for queue writes (avoids issues with providers that drop idle connections); a single long-lived connection is fine and required for the Worker side (BullMQ workers need `maxRetriesPerRequest: null`).

### Twilio webhook handling
Every Twilio-facing webhook route (voice TwiML, status callbacks, recording callbacks, inbound calls) must be public (no JWT — Twilio can't send one) and must **always return HTTP 200 with valid TwiML** on error, never a JSON error or non-200 status — a non-200/malformed response makes Twilio play a generic "application error" tone to a live caller. Build a small fallback-TwiML helper (`<Say>...</Say><Hangup/>`) and use it in every catch block on these routes. **Implement Twilio request-signature validation (`X-Twilio-Signature`) on these routes** — this was missing in the reference implementation and should be treated as a required security control in the rebuild, not optional.

## 5. Storage & email provider decisions (do this from the start, don't retrofit)

- **Email**: use **AWS SES**, not any third-party transactional email SaaS. Build one `sendMail({to, subject, html, replyTo?})` function as the single choke point every feature calls through (OTP emails, password reset, any notification emails) — this keeps the provider swappable later without touching call sites.
- **File storage** (avatars, any uploaded documents, generated PDFs/receipts if you build anything like that): use **AWS S3**, not Cloudinary or local disk. Same pattern — one small storage-service module (`uploadBuffer(buffer, folder, filename) → url`) that every upload feature calls through.
- **TTS audio files**: for any provider that needs pre-generated audio files (rather than native Twilio `<Say>` support), either serve them from S3 or regenerate on demand per-request rather than writing to local disk and assuming the same server instance serves them back — required given the multi-instance scale target in §2.

## 6. Frontend conventions (apply to every frontend section)

- **Auth**: store `{accessToken, refreshToken, user}` in `localStorage`. Axios instance with a request interceptor attaching `Authorization: Bearer <token>` and a response interceptor that does single-flight token refresh on 401 (queue concurrent requests while a refresh is in-flight, replay them once it resolves).
- **Routing**: React Router v6, every page lazy-loaded, one top-level `<Suspense>` with a shared page-loading fallback. A single `Private` guard component wraps every authenticated route (redirect to `/login` if not authenticated) — no role branching needed, there's only one role.
- **Layout shell**: fixed collapsible sidebar (data-driven from one nav-config array, not hardcoded per-page) + top navbar (page title, theme toggle, notifications bell, profile menu) + a content `<Outlet/>`.
- **Design system**: Tailwind, `darkMode:'class'`, dark by default, theme persisted to `localStorage` and applied via a synchronous pre-hydration script in `index.html` (no flash of wrong theme). Primary accent: a violet scale (`#7c3aed` as the base). Define real named CSS custom properties for surface colors (card/elevated/border) usable in both light and dark — don't hardcode hex values per-component (a gap in the reference implementation worth actually fixing here).
- **State management** — pick one approach and use it consistently across every page, don't mix: either commit fully to TanStack React Query for all server state (recommended — gives you caching/dedup for free, which matters more at this data scale), or use a consistent plain-fetch-with-hooks pattern everywhere. Do not build some pages with React Query and others with ad hoc `useState`/`useEffect` fetches.
- **Shared UI primitives**: build once and reuse everywhere — a `Modal` component (don't let every page hand-roll its own portal-based modal), a `Badge` component with status→color mapping (don't redefine per page), a `Pagination` component, a `Select`/searchable-dropdown component. Consistency here is a design-quality bar for the rebuild, not a nice-to-have.
- **Dashboard/analytics pages**: never fetch "all rows across N pages and reduce client-side" for anything analytics-shaped — at this platform's call volume that pattern is broken, not just slow. Build real backend aggregation endpoints (MongoDB `$group` pipelines) for any dashboard/report page.

## 7. Consolidated environment variables

```
NODE_ENV, PORT, ROLE (api|worker|all)
MONGODB_URI
REDIS_URL
JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, JWT_ACCESS_EXPIRES_IN (default 2h), JWT_REFRESH_EXPIRES_IN (default 7d)
FRONTEND_URL, PUBLIC_APP_URL
OPENAI_API_KEY, ANTHROPIC_API_KEY
TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_API_KEY, TWILIO_API_SECRET, TWILIO_PHONE_NUMBER, TWILIO_WEBHOOK_BASE_URL
DEEPGRAM_API_KEY
CARTESIA_API_KEY, CARTESIA_CONCURRENT_LIMIT
ELEVENLABS_API_KEY
AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION
AWS_SES_FROM_EMAIL (email sending — see §5)
AWS_S3_BUCKET (file storage — see §5)
GOOGLE_PLACES_API_KEY (only if you keep any business-lookup feature)
VITE_API_URL, VITE_DEV_PROXY_TARGET (frontend build-time)
```

## 8. Recommended build order

1. **Auth** (backend + frontend shell/routing/auth pages together) — everything else depends on it.
2. **Agents** (backend + frontend) — the core configuration object every call/campaign/workflow references.
3. **Real-time call pipeline** (backend) — the most complex piece; get one working call path (outbound, one voice provider, no barge-in) before adding streaming/barge-in/multi-provider fallback.
4. **Calls/Chats/Reports** (frontend) — view what the pipeline produces.
5. **Contacts & Leads** (backend + frontend) — simple CRUD, needed by Campaigns.
6. **Campaigns** (backend + frontend) — depends on Agents + Contacts/Leads + the call pipeline.
7. **Workflows** (backend + frontend) — depends on the call pipeline's trigger points.
8. **Knowledge Base** (backend + frontend) — can be built in parallel with the above; only wire into the call pipeline's system-prompt assembly once both exist.
9. **Dashboard** (frontend) — build its backend aggregation endpoint alongside it, not after.
10. **Settings, Notifications, Voices, shared components/design system** — round out the rest; some of this (design system primitives) should really be extracted *as you go* in steps 1–9, not bolted on at the end.

Each of these has its own detailed prompt file in this folder (`backend/NN-*.md`, `frontend/NN-*.md`) — follow this build order and this master guide together.
