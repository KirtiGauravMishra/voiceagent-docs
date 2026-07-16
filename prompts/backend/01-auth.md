# Backend Prompt 01 — Auth & Users

> Read `../00-master-rebuild-guide.md` first. This prompt builds the authentication system every other backend module depends on.

## Goal
Build a JWT (access + refresh) authentication system with mandatory email verification, optional TOTP 2FA, DB-backed revocable sessions, and a long-lived API-key path for programmatic access. Single role (`user`) — no role hierarchy, no admin/superadmin/employee roles.

## Data models

**`User`**
| Field | Type | Rules |
|---|---|---|
| `name` | string | required, max 100 |
| `email` | string | required, unique, lowercased |
| `passwordHash` | string | bcrypt, cost 10; never returned by `toJSON()` |
| `avatarUrl` | string | default `''` |
| `suspended` | boolean | default `false` — blocks login/refresh/every authenticated request when true |
| `isEmailVerified` | boolean | default `false` |
| `emailVerificationOtp` | string, `select:false` | bcrypt-hashed 6-digit code |
| `emailVerificationExpires` | Date, `select:false` | now + 15 min |
| `passwordResetToken` | string, `select:false` | bcrypt-hashed 6-digit code |
| `passwordResetExpires` | Date, `select:false` | now + 30 min |

`toJSON()` transform: `_id`→`id`, strip `__v` and `passwordHash`.

**`RefreshToken`** — one row per active session/device (not one per user).
| Field | Type |
|---|---|
| `token` | string, unique — the signed refresh JWT itself |
| `userId` | ObjectId → User |
| `expiresAt` | Date |

Add a MongoDB TTL index `{expiresAt:1}, {expireAfterSeconds:0}` so expired rows self-delete — no cleanup job needed. This collection **is** the "active sessions" list shown in account settings.

**`ApiKey`**
| Field | Type |
|---|---|
| `ownerId` | ObjectId → User, indexed |
| `name` | string, max 120 |
| `keyPrefix` | string — first 8 + last 4 chars of the plaintext, for display only |
| `keyHash` | string, unique — `sha256(plaintext)`, never store the plaintext |
| `createdByEmail` | string |
| `lastUsedAt` | Date |
| `status` | enum `active\|inactive`, indexed |

Key format: `ap-<48 hex chars from crypto.randomBytes(24)>`. Return the plaintext key **exactly once**, in the create-response — never retrievable again. Hash with plain SHA-256 (no salt needed — the input already has 192 bits of entropy).

**`UserSettings`** (2FA fields only here — rest of this model belongs to the Settings module)
| Field | Type |
|---|---|
| `twoFactorEnabled` | boolean, default `false` |
| `totpSecret` | string, `select:false` — set only once setup is confirmed |
| `totpPendingSecret` | string, `select:false` — holds the secret between "start setup" and "confirm first code" |

## Middleware

**`authenticate`** (applied to every non-public router):
1. Read `Authorization: Bearer <token>`, fall back to `X-API-Key` header. No token → 401.
2. If the token starts with `ap-`: hash it, look up `ApiKey` where `status:'active'`. 401 if not found. Load the owning `User`; 403 if `suspended`. Fire-and-forget update `lastUsedAt`. Set `req.user` from the **live DB user** (fresh role/email every request).
3. Otherwise: verify as a JWT. Payload already carries `{id, email, name}`. Still do one DB read for `suspended` status (fail **open** on a DB error — log and let the request through — fail **closed** only on a definitive `suspended:true`). Set `req.user` from the JWT payload.

Since there's only one role, skip building a `requireRole` layer — every protected resource is scoped by `ownerId === req.user.id` at the query level instead.

## Token design

| Token | TTL | Payload | Notes |
|---|---|---|---|
| Access | `2h` default | `{id, email, name}` | `Authorization: Bearer` on every call |
| Refresh | `7d` default | `{id, email, name}` | Exchanged at `/api/auth/refresh`; persisted in `RefreshToken` so it's revocable server-side |
| 2FA-pending | `10m`, hardcoded | `{id, typ:'2fa_pending'}` | Bridges password-check → TOTP-check; verify function must reject any token missing `typ==='2fa_pending'` so it can never be replayed as a real access token even though it shares the signing secret |

## Flows to implement

**Register** (`POST /api/auth/register {name, email, password}`): password policy ≥8 chars + upper + lower + digit, enforced both in a Yup schema at the route and again in the service. 409 if email exists. Hash password, create user with `isEmailVerified:false`. Generate 6-digit OTP, bcrypt-hash into `emailVerificationOtp`, 15-min expiry. Email the **plaintext** OTP via SES. Registration succeeds even if the email fails to send (log it — user can trigger a resend by attempting login). Response: `{requiresEmailVerification:true, email}`, HTTP 201, no tokens yet.

**Verify email** (`POST /api/auth/verify-email {email, code}`): 404 if no user, 409 if already verified, 401 if OTP missing/expired/mismatched. On success: mark verified, clear OTP fields, **issue a real session immediately** (verifying email logs you in).

**Login** (`POST /api/auth/login {email, password}`), branch in order:
1. User not found or password mismatch → `401 Invalid email or password` (identical message either way — no user-enumeration).
2. `suspended` → 403.
3. Not verified → generate+email a fresh OTP (this doubles as the "resend" mechanism — no separate resend endpoint), return `{requiresEmailVerification:true, email}`.
4. 2FA enabled → return `{requiresTwoFactor:true, twoFactorToken}` (the 10-min pending token). No session yet.
5. Otherwise → issue a full session (`{user, accessToken, refreshToken}`), persist the refresh row.

Log a security-audit event (`login_success`/`login_failed`) with IP + User-Agent, fire-and-forget.

**Complete 2FA** (`POST /api/auth/login/2fa {twoFactorToken, code}`): verify the pending token (throw a distinct "session expired, start over" error vs. "wrong code, retry"). Re-check `suspended`. Re-verify 2FA is still enabled (handles a disable-mid-flow race). Check the TOTP code. Issue a full session.

**Refresh** (`POST /api/auth/refresh {refreshToken}`): look up the stored row, 401 if missing/expired. If the user is now suspended, delete **all** their refresh tokens (kill every session) and 403. Verify the JWT signature. **Rotate**: delete the old row, issue + store a brand-new access+refresh pair. The old refresh token must 401 if reused.

**Logout** (`POST /api/auth/logout {refreshToken}`): delete exactly that one row. No access token required — this must work even with an expired access token, using only the refresh token.

**Forgot password** (`POST /api/auth/forgot-password {email}`): always respond success-shaped even for an unknown email (no account-enumeration via status code). Generate OTP → `passwordResetToken`, 30-min expiry, email it via SES. Don't leak whether email sending is configured.

**Reset password** (`POST /api/auth/reset-password {email, otp, password}`): same password policy. 401 if token missing/expired/mismatched. On success: set new password hash, clear reset fields, **delete every RefreshToken for that user** (resetting your password logs you out everywhere).

**2FA setup** (under Settings, requires an existing session — `POST /api/settings/account/2fa/setup`, `/verify`, `/disable`):
- `setup` (no body): reject if 2FA already enabled. Generate a base32 secret, store as `totpPendingSecret`, build an `otpauth://` URI, render a QR code data-URL (200px). Return `{secret, qrDataUrl, otpauthUrl}`.
- `verify {code}`: check against `totpPendingSecret`; on match, promote to `totpSecret`, set `twoFactorEnabled:true`, clear the pending field.
- `disable {password, code}`: require **both** the current password and a valid current TOTP code (two factors to turn off two factors). Clear all 2FA fields.

**Sessions** (`GET /api/settings/account/sessions`, `POST /api/settings/account/sessions/:id/revoke`): list every `RefreshToken` row for the user, newest first; revoke deletes one row scoped to the owner (404 if not found/not yours).

**API keys** (`GET/POST /api/settings/api-keys`, `DELETE /api/settings/api-keys/:id`): standard CRUD. `create` is the only place the plaintext key is ever returned.

## Security audit log
Create a `SecurityAuditLog` model and a `logSecurityEvent({event, req, userId?, email?, success?, meta?})` helper that captures event name, client IP (`X-Forwarded-For` first segment, else socket address), User-Agent, success flag. Call it fire-and-forget (never let a logging failure break an auth flow) from: `login_success`, `login_failed`, `2fa_verified`, `logout`, `password_reset_request`, `password_reset_complete`, `2fa_setup`, `2fa_disabled`.

## API reference

| Method | Path | Auth | Rate limit | Purpose |
|---|---|---|---|---|
| POST | `/api/auth/register` | public | 5/hour/IP | create account + send OTP |
| POST | `/api/auth/login` | public | 25/15min/IP | password check |
| POST | `/api/auth/login/2fa` | public | 25/15min/IP | complete 2FA |
| POST | `/api/auth/verify-email` | public | 25/15min/IP | confirm OTP |
| POST | `/api/auth/refresh` | public | 30/15min/IP | rotate tokens |
| POST | `/api/auth/logout` | public | — | delete one session |
| POST | `/api/auth/forgot-password` | public | 25/15min/IP | email reset OTP |
| POST | `/api/auth/reset-password` | public | — | consume OTP, kill all sessions |
| GET/POST/DELETE | `/api/settings/api-keys[/:id]` | JWT | — | API key CRUD |
| POST | `/api/settings/account/2fa/{setup,verify,disable}` | JWT | — | 2FA lifecycle |
| GET/POST | `/api/settings/account/sessions[/:id/revoke]` | JWT | — | session list/revoke |

Response envelope and error classes: see master guide §4.

**Rate limiting**: given the 100k+/day, 1,000+-concurrent scale target, do **not** use a plain in-process rate limiter Map for these limits — back it with Redis (e.g. a sliding-window counter) so limits are consistent across every horizontally-scaled instance, not reset per-process.

## Environment variables
`JWT_ACCESS_SECRET` (≥32 chars), `JWT_REFRESH_SECRET` (≥32 chars, different from access), `JWT_ACCESS_EXPIRES_IN` (default `2h`), `JWT_REFRESH_EXPIRES_IN` (default `7d`), `MONGODB_URI`, plus SES config from the master guide (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_SES_FROM_EMAIL`) for OTP emails.

## Acceptance checklist
- [ ] Register → verify-email → session issued → protected route works with the access token.
- [ ] Login with 2FA enabled returns the pending-token branch, not a session; completing 2FA issues a session.
- [ ] Refresh rotates the token — reusing an old refresh token after rotation returns 401.
- [ ] Password reset invalidates every existing session for that user.
- [ ] Suspending a user immediately blocks their API-key requests and blocks JWT requests on the next request past token issuance (fails open only on a genuine DB error, not on suspension).
- [ ] An API key's plaintext is visible exactly once, at creation.
- [ ] All rate limits are enforced consistently across multiple instances (Redis-backed), not per-process.
