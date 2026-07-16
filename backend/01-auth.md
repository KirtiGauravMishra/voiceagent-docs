# 01 — Auth & Users

> Backend domain doc. Source of truth: `backend/src/{controllers,services,routes,middlewares,models,utils,validations}` as of the current `main` branch.

## 1. Overview

Authentication is a custom JWT (access + refresh token) system backed by MongoDB, with:

- Email/password login with bcrypt hashing.
- Mandatory email verification via a 6-digit OTP before first login succeeds.
- Optional TOTP-based two-factor authentication (Google Authenticator–style, via `@otplib/preset-default`).
- DB-backed refresh tokens (allows server-side session revocation / "log out other devices").
- A separate long-lived **API key** auth path (`ap-...` tokens) for programmatic/API access, checked by the same middleware as JWTs.
- A `role` field on the schema, scoped to `user` for the main product surface documented here (see §3).

There is no OAuth/social login and no cookie-based session — the frontend is a pure Bearer-token SPA client (see `frontend` docs).

## 2. Data Models

### `User` (`backend/src/models/User.model.ts`)

| Field | Type | Notes |
|---|---|---|
| `name` | string | required, max 100 |
| `email` | string | required, unique, lowercased — unique index comes from `unique: true` on the field (do not add a duplicate `.index()` call) |
| `passwordHash` | string | bcrypt hash, cost factor 10; stripped from all `toJSON()` output |
| `avatarUrl` | string | default `''` |
| `billingAddress` | string | default `''` |
| `role` | enum | scoped to `'user'` for the main product surface documented here, default `'user'` |
| `suspended` | boolean | default `false` — hard-blocks login, refresh, and every authenticated request once true |
| `isEmailVerified` | boolean | default `false` |
| `emailVerificationOtp` | string (select:false) | bcrypt-hashed 6-digit OTP |
| `emailVerificationExpires` | Date (select:false) | 15 minutes from issue |
| `passwordResetToken` | string (select:false) | bcrypt-hashed 6-digit OTP |
| `passwordResetExpires` | Date (select:false) | 30 minutes from issue |

`toJSON()` transform renames `_id` → `id`, strips `__v` and `passwordHash` on every serialization — controllers never need to manually scrub the password hash.

### `RefreshToken` (`backend/src/models/RefreshToken.model.ts`)

One document per active session (not one per user — logging in from N devices creates N rows).

| Field | Type | Notes |
|---|---|---|
| `token` | string | the actual signed refresh JWT, unique |
| `userId` | ObjectId → User | |
| `expiresAt` | Date | mirrors the JWT's own expiry |

A MongoDB **TTL index** (`{ expiresAt: 1 }, { expireAfterSeconds: 0 }`) auto-deletes expired rows — no manual cleanup job needed. This collection doubles as the "sessions" list shown in account settings (`GET /api/settings/account/sessions`).

### `ApiKey` (`backend/src/models/ApiKey.model.ts`)

| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `name` | string | user-supplied label, max 120 |
| `keyPrefix` | string | first 8 + last 4 chars of the plaintext key, shown in UI so the user can identify a key without ever seeing it again |
| `keyHash` | string | `sha256(plaintext)`, unique — the plaintext is **never stored** |
| `createdByEmail` | string | denormalized owner email at creation time |
| `lastUsedAt` | Date | best-effort, updated fire-and-forget on each authenticated request |
| `status` | enum | `'active' \| 'inactive'`, indexed |

Key generation (`generateApiKey()`): `ap-<48 hex chars from crypto.randomBytes(24)>`. The plaintext is returned **once**, at creation time, in the API response — it cannot be retrieved again afterward. `hashApiKey()` (SHA-256, no salt — acceptable here because the input itself is 192 bits of CSPRNG entropy, unlike a user password) is used both to store and to look up keys.

### `UserSettings` (2FA-relevant fields only — see `08-notifications-settings.md` for the rest)

| Field | Type | Notes |
|---|---|---|
| `twoFactorEnabled` | boolean | default `false` |
| `totpSecret` | string (select:false) | base32 TOTP shared secret, set only once 2FA setup is confirmed |
| `totpPendingSecret` | string (select:false) | holds the secret between "start setup" (QR shown) and "verify setup" (first code confirmed) — discarded either way after verify/cancel |

## 3. Authorization Model

Every account relevant to this documentation set is a normal customer/tenant `user`. Everything under `/api/agents`, `/api/calls`, `/api/billing`, etc. is scoped to `req.user.id` as the owner — a flat, single-role model for the main product surface.

`requireRole(...roles)` is a flat `roles.includes(req.user.role)` check (not a ranked hierarchy) — not used anywhere on the main-app routes covered in this doc set.

### Middleware (`backend/src/middlewares/auth.middleware.ts`)

**`authenticate`** — runs on every protected router via `router.use(authenticate, ...)`:

1. Reads `Authorization: Bearer <token>` header, or falls back to `X-API-Key` header.
2. No token → `401 Access token required`.
3. **If token starts with `ap-`** → API-key path: hash it, look up `ApiKey` where `status:'active'`; 401 if not found. Load the owning `User`, 403 if `suspended`. Fire-and-forget bump `lastUsedAt`. Populate `req.user = { id, email, name, role }` from the **live DB user**, then `next()`.
4. **Otherwise** → JWT path: `verifyAccessToken(token)` (throws → 401 `Invalid or expired access token`). Payload already contains `{id, email, name, role}` so no DB round-trip is strictly required for authorization — but the middleware still does one extra DB read (`User.findById(payload.id).select('suspended')`) purely to catch suspensions that happened *after* the token was issued. This check **fails open** on a DB error (logs a warning, lets the request through trusting the JWT) and only blocks on a definitive `suspended: true`. `req.user = payload` (the JWT claims, not a fresh DB read), then `next()`.

**`requireRole(...roles)`** — placed after `authenticate` on a router: `403 Forbidden` unless `req.user.role` is literally one of the given strings. Not used on any of the main-product routes covered in this documentation set — those are all `authenticate`-only, scoped by `req.user.id`.

### Important asymmetry to know when rebuilding

The **API-key path re-derives role/email from a fresh DB read every request**, while the **JWT path trusts the token's embedded role/email and only re-checks `suspended`**. This means: if you change a user's `role` in the database, JWT-authenticated sessions keep acting on the *old* role until their access token expires (default 2h) or they re-login; API-key-authenticated requests see the change immediately.

## 4. Token Design (`backend/src/utils/jwt.ts`)

| Token | Signed with | Default TTL | Payload | Purpose |
|---|---|---|---|---|
| Access token | `JWT_ACCESS_SECRET` | `2h` (`JWT_ACCESS_EXPIRES_IN`) | `{id, email, name, role}` | Sent as `Authorization: Bearer` on every API call |
| Refresh token | `JWT_REFRESH_SECRET` | `7d` (`JWT_REFRESH_EXPIRES_IN`) | `{id, email, name, role}` | Exchanged at `/api/auth/refresh`; also persisted as a `RefreshToken` row so it can be revoked server-side |
| 2FA-pending token | `JWT_ACCESS_SECRET` (same secret, different `typ` claim) | `10m` (hardcoded) | `{id, typ:'2fa_pending'}` | Issued by `/login` when 2FA is required; bridges password-check → TOTP-check without granting a real session. `verifyTwoFactorPendingToken` throws unless `typ === '2fa_pending'`, so it can never be replayed as a normal access token even though it shares the signing secret. |

`parseTtlMs("7d")` parses `\d+[smhd]` strings into milliseconds — used to compute `RefreshToken.expiresAt` explicitly (the JWT's own `exp` claim is the enforcement mechanism at verify-time; the DB `expiresAt` + TTL index is what actually deletes the row and is what `refresh()` checks first).

## 5. Flows

### 5.1 Registration → Email verification

```
POST /api/auth/register  { name, email, password }
```
1. Password policy enforced in the **service** layer (not just validation): ≥8 chars, at least one upper, one lower, one digit. (The Yup schema at the route layer enforces the same rule at the HTTP boundary — see §7 — so this is defense-in-depth, not the only check.)
2. 409 if email already registered.
3. `bcrypt.hash(password, 10)`, create `User` with `isEmailVerified:false`.
4. Generate a 6-digit numeric OTP, bcrypt-hash it into `emailVerificationOtp`, set `emailVerificationExpires = now + 15min`.
5. Email the **plaintext** OTP (never the hash) via `sendVerificationMail` → `mailer.service.sendMail`. If email send fails, registration still succeeds (logged as an error) — the user can trigger a re-send by attempting to log in.
6. Response: `{ requiresEmailVerification: true, email }` (HTTP 201). No tokens issued yet.

```
POST /api/auth/verify-email  { email, code }
```
- 404 if no such user; 409 if already verified.
- 401 if OTP missing/expired or doesn't bcrypt-compare.
- On success: `isEmailVerified = true`, clears the OTP fields, fires `fireOnSignupRules(...)` (subscription-automation "on signup" email rule — fire-and-forget, not awaited), and **immediately issues a real session** (access + refresh token) — verifying email logs the user in.

### 5.2 Login

```
POST /api/auth/login  { email, password }
```
Branches, in order:
1. User not found or password mismatch → `401 Invalid email or password` (same message either way — no user-enumeration leak).
2. `suspended` → `403`.
3. `!isEmailVerified` → generates and emails a **fresh** OTP (overwriting any previous one/expiry), returns `{ requiresEmailVerification: true, email }`. This is also how a user "resends" a verification code — there's no separate resend endpoint, they just attempt login again.
4. `UserSettings.twoFactorEnabled && totpSecret` set → returns `{ requiresTwoFactor: true, twoFactorToken }` (the 10-minute pending token). No session yet.
5. Otherwise → `createSessionForUser(user)`: signs access+refresh, persists the refresh row, returns `{ user, accessToken, refreshToken }`.

The controller (`auth.controller.ts`) logs a `login_success`/`login_failed` row to `SecurityAuditLog` via `logSecurityEvent` (fire-and-forget) around the service call — capturing IP and User-Agent from the request.

Login timing is logged at each stage (`findUser`, `bcryptCompare`, `tokenCreate`/`twoFactorStep`, `total` in ms) — useful for diagnosing slow-login incidents (e.g. bcrypt cost vs. Mongo connection latency).

### 5.3 Two-factor completion

```
POST /api/auth/login/2fa  { twoFactorToken, code }
```
1. Verify the pending token (throws → `401 Invalid or expired login session`, distinct message from a bad TOTP code, so the frontend can tell "start over" apart from "wrong code, retry").
2. Re-check `suspended` (the account could have been suspended in the window between password step and 2FA step).
3. Re-fetch `UserSettings` and re-verify 2FA is still enabled (defends against a race where 2FA was disabled mid-flow).
4. `assertTotpCodeForUser(secret, code)` — TOTP window check via `@otplib/preset-default`'s `authenticator.check`.
5. On success → `createSessionForUser(user)` (a full session, same as a normal login).

### 5.4 Refresh (token rotation)

```
POST /api/auth/refresh  { refreshToken }
```
1. Look up the token row; 401 if missing or `expiresAt < now`.
2. Re-check the owning user isn't suspended; if it is, **delete all refresh tokens for that user** (kills every session, not just this one) and 403.
3. Verify the JWT signature itself (`verifyRefreshToken`) — note this happens *after* the DB lookup, so a DB hit occurs even for a syntactically-invalid token string, as long as it happens to match a stored `token` value (it won't, in practice, since only valid signed tokens are ever stored — but a reviewer should know the order isn't "verify signature first").
4. **Rotates**: deletes the old `RefreshToken` row, issues a brand-new access+refresh pair, persists the new refresh row. The old refresh token is now dead — reusing it 401s. This is standard refresh-token rotation (limits the blast radius of a leaked refresh token to one use).

### 5.5 Logout

```
POST /api/auth/logout  { refreshToken }
```
Deletes exactly the one matching `RefreshToken` row (does not affect other devices/sessions). No auth header required — logout works with just the refresh token, mirroring how refresh itself works, so a client can log out even with an expired/missing access token.

### 5.6 Forgot / Reset password

```
POST /api/auth/forgot-password  { email }
```
- Always returns `{ notFound: false }`-shaped success even if the email doesn't exist (`{ notFound: true }` is returned internally but the controller currently surfaces it — check whether the frontend actually branches on this; either way no 404 is thrown, avoiding account enumeration via error status).
- Generates a 6-digit OTP, bcrypt-hashes into `passwordResetToken`, 30-minute expiry.
- If the mail provider isn't configured (`sendMail` returns `{skipped:true}`, e.g. Brevo env vars absent), logs a warning but still returns success (doesn't leak provider configuration state to the client).

```
POST /api/auth/reset-password  { email, otp, password }
```
- Same password policy as registration.
- 401 if no reset token on file, expired, or doesn't match.
- On success: hashes and sets the new password, clears reset-token fields, and **deletes every `RefreshToken` for that user** — resetting your password logs you out everywhere.

### 5.7 Two-Factor Setup / Disable (lives under Settings, not Auth)

Mounted at `/api/settings/account/2fa/*` (JWT-protected, i.e. you must already be logged in — this is not part of the login flow, it's how an already-authenticated user turns 2FA on/off).

```
POST /api/settings/account/2fa/setup     (no body)
```
`twoFactor.service.startTotpSetup(ownerId, email)` — 409-equivalent (`ValidationError`) if 2FA is already enabled (must disable first to re-setup). Generates a new base32 secret via `authenticator.generateSecret()`, stores it as `totpPendingSecret` (upsert), builds an `otpauth://` URI (`authenticator.keyuri(email, issuer, secret)`, issuer = `TOTP_APP_NAME` env or falls back to `"VoiceAgent"`), and renders it to a QR data-URL (`qrcode`, 200px). Returns `{ secret, qrDataUrl, otpauthUrl }` — the frontend shows the QR (and the raw secret as a manual-entry fallback).

```
POST /api/settings/account/2fa/verify    { code }
```
Validates `code` against `totpPendingSecret`; on match, promotes it to `totpSecret`, sets `twoFactorEnabled:true`, clears the pending field. Logs a `2fa_setup` security-audit event.

```
POST /api/settings/account/2fa/disable   { password, code }
```
Requires **both** the current account password (bcrypt-compared) **and** a valid current TOTP code — disabling 2FA can't be done with just one factor. Clears `twoFactorEnabled`, `totpSecret`, `totpPendingSecret`. Logs a `2fa_disabled` event.

### 5.8 Sessions & API Keys (also under Settings)

```
GET  /api/settings/account/sessions
```
Lists every `RefreshToken` row for the user (i.e. every device/browser currently able to silently refresh), newest first. `device`/`location` are currently **placeholder strings** (`"Current Web Session"` / `"Web Session"` / `"Unknown"`) — the schema doesn't actually capture User-Agent or IP per refresh token today, so this is a display stub, not real device fingerprinting. `current` is determined by matching the `X-Refresh-Token` request header (if the frontend sends it) against each row's token, falling back to "first row = current" if that header isn't sent.

```
POST /api/settings/account/sessions/:id/revoke
```
Deletes one `RefreshToken` row by its Mongo `_id` (scoped to `ownerId` so you can't revoke someone else's session by guessing an ID). 404 if not found/not yours.

```
GET    /api/settings/api-keys
POST   /api/settings/api-keys   { name }
DELETE /api/settings/api-keys/:id
```
Standard CRUD over `ApiKey`. `createApiKey` is the **only** place the plaintext key is ever visible — response includes `key: generated.plain` once; `listApiKeys` only ever returns `keyPrefix`. Both list/create routes are additionally gated by `assertApiAccess(req.user.id)` (a plan-feature check — see billing docs — API-key access can be a paid-plan-gated feature; not relevant if billing is stripped for the in-house rebuild, in which case this call should simply be removed/no-op'd).

## 6. Security Audit Log

`services/securityAudit.service.ts` → `logSecurityEvent({ event, req, userId?, email?, success?, meta? })` writes a row to `SecurityAuditLog` capturing: event name, resolved client IP (`X-Forwarded-For` first segment, else socket remote address), `User-Agent`, success flag, optional metadata. Every call site wraps this in `void logSecurityEvent(...)` (fire-and-forget — a logging failure must never break the auth flow) and the function itself swallows/logs its own errors.

Events currently emitted from the auth surface: `login_success`, `login_failed`, `2fa_verified` (both success/fail), `logout`, `password_reset_request`, `password_reset_complete`, `2fa_setup`, `2fa_disabled`.

## 7. API Reference

### Auth router — `/api/auth` (all public, no `authenticate` required)

| Method | Path | Rate limit | Validation schema | Purpose |
|---|---|---|---|---|
| POST | `/register` | 5 / hour / IP | `registerSchema` | Create account, send verification OTP |
| POST | `/login` | 25 / 15 min / IP | `loginSchema` | Password check → session, or 2FA/verify-email branch |
| POST | `/login/2fa` | 25 / 15 min / IP | `loginTwoFactorSchema` | Complete login after TOTP challenge |
| POST | `/verify-email` | 25 / 15 min / IP | `verifyEmailSchema` | Confirm OTP, activates account + issues session |
| POST | `/refresh` | 30 / 15 min / IP | `refreshSchema` | Rotate access+refresh token pair |
| POST | `/logout` | — (unlimited) | `refreshSchema` | Delete one session |
| POST | `/forgot-password` | 25 / 15 min / IP | `forgotPasswordSchema` | Email a reset OTP |
| POST | `/reset-password` | — (unlimited) | `resetPasswordSchema` | Consume OTP, set new password, kill all sessions |

Rate limiters are a custom in-memory implementation (`middlewares/rateLimiter.middleware.ts`, `inMemoryLimiter(windowMs, max, name)`) — **not** Redis-backed, so limits reset on process restart and are **per-process**, not shared across horizontally-scaled instances. Worth revisiting if the rebuild runs multiple Replit instances behind a load balancer.

### Settings router (auth-adjacent endpoints) — `/api/settings`, JWT required

| Method | Path | Purpose |
|---|---|---|
| GET | `/api-keys` | List your API keys (prefix only) |
| POST | `/api-keys` | Create a key (returns plaintext once) |
| DELETE | `/api-keys/:id` | Revoke a key |
| POST | `/account/2fa/setup` | Begin TOTP enrollment, returns QR |
| POST | `/account/2fa/verify` | Confirm first TOTP code, enables 2FA |
| POST | `/account/2fa/disable` | Requires password + TOTP code |
| GET | `/account/sessions` | List active refresh-token sessions |
| POST | `/account/sessions/:id/revoke` | Kill one session |

### Request/response shape conventions (apply platform-wide)

Success: `{ "success": true, "data": <payload> }` (`utils/response.ts::sendSuccess`), or `{ success, data, pagination:{page,limit,total,totalPages} }` for list endpoints (`sendPaginated`).

Errors: thrown as one of `AppError` subclasses (`utils/errors.ts`) — `NotFoundError`(404,`NOT_FOUND`), `UnauthorizedError`(401,`UNAUTHORIZED`), `ForbiddenError`(403,`FORBIDDEN`), `ConflictError`(409,`CONFLICT`), `ValidationError`(422,`VALIDATION_ERROR`, optional `details`) — caught by the global `error.middleware.ts` and rendered as `{ success:false, error:{ code, message, details? } }` (verify exact shape in `middlewares/error.middleware.ts` when writing the bootstrap doc).

Sample login response (2FA-off path):
```json
{
  "success": true,
  "data": {
    "user": { "id": "665f...", "name": "Jane Doe", "email": "jane@x.com", "role": "user", "isEmailVerified": true, "suspended": false, "avatarUrl": "", "billingAddress": "", "createdAt": "...", "updatedAt": "..." },
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "eyJhbGciOi..."
  }
}
```

## 8. Rebuild Notes (Replit target)

### Environment variables required

| Variable | Required | Notes |
|---|---|---|
| `JWT_ACCESS_SECRET` | yes | ≥32 chars, HMAC signing key for access tokens |
| `JWT_REFRESH_SECRET` | yes | ≥32 chars, **must differ** from the access secret |
| `JWT_ACCESS_EXPIRES_IN` | no | default `2h` |
| `JWT_REFRESH_EXPIRES_IN` | no | default `7d` |
| `MONGODB_URI` | yes | Users/RefreshToken/ApiKey/UserSettings/SecurityAuditLog all live here |
| Email sending vars | yes | see below — needed for verification OTP, password-reset OTP |

### Dependencies

`bcryptjs`, `jsonwebtoken`, `@otplib/preset-default`, `qrcode`, `mongoose`, `yup` (route-layer validation). No changes needed for an in-house/Replit rebuild — all pure npm packages, no external auth SaaS.

### Email provider swap (Brevo → AWS SES)

The current code sends verification/reset OTP emails through `services/mailer.service.ts`, which is Brevo-backed (`BREVO_SMTP_USER`/`BREVO_SMTP_KEY`/`BREVO_API_KEY`). Per the in-house rebuild plan, this should be swapped for **AWS SES** (the team already has an SES-verified sending identity and an S3 bucket for storage — S3 is not relevant to auth, but SES is a straight drop-in for `sendMail()`'s call sites). Concretely, `sendMail({to, subject, html})` in `mailer.service.ts` is the single choke point every auth flow calls through (`sendVerificationMail`, `forgotPassword`) — swapping its implementation to `@aws-sdk/client-ses`'s `SendEmailCommand` (or `SendRawEmailCommand` if HTML+attachments are ever needed) requires no changes to `auth.service.ts` itself, only to the mailer implementation and its env vars (`AWS_SES_REGION`, `AWS_SES_FROM_EMAIL`, plus the already-present `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`).

### Hardcoded values to review before rebuild

- `TOTP_APP_NAME` issuer fallback of `"VoiceAgent"` (`twoFactor.service.ts:8`) — cosmetic (shows up as the entry's label in authenticator apps), update to match the in-house product name.
- In-memory rate limiters are single-process — fine for a single Replit deployment, but note if you ever scale to multiple instances.

### Setup steps for a fresh Replit environment

1. Provision MongoDB (Atlas or self-hosted) and set `MONGODB_URI`.
2. Generate two independent 32+ char random secrets for `JWT_ACCESS_SECRET`/`JWT_REFRESH_SECRET`.
3. Configure AWS SES sending identity + region/credentials for OTP emails (see above).
4. No Redis dependency for auth itself (Redis is used elsewhere — jobs/quota — not by this domain).
