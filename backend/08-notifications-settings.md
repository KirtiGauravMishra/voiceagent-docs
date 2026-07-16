# 08 — Notifications, Provider Connections, API Keys & Account Settings

> Backend domain doc. Source: `backend/src/{models/{Notification,ProviderConnection}.model.ts, controllers/{notification,settings}.controller.ts, routes/{notification,settings}.routes.ts, services/settings.service.ts}`. ApiKey/2FA/session models and flows are documented in full in `01-auth.md`; this file covers the rest of the Settings surface plus in-app Notifications.

## 1. Notifications (`models/Notification.model.ts`)

The in-app notification/announcement system — surfaces things like "call completed" events or platform-wide announcements.

| Field | Type | Notes |
|---|---|---|
| `title` / `message` | string | max 200 / 2000 |
| `type` | enum | `info \| warning \| success \| alert` — drives the icon/color in the UI |
| `target` | enum | `all` (every user sees it) or `specific` (only `targetUserId`) |
| `targetUserId` | ObjectId → User | only set when `target:'specific'` |
| `readBy` | ObjectId[] | every user who has marked it read — **a single doc serves all recipients**, "read" state is per-user via array membership, not one row per user |
| `createdBy` | ObjectId → User | optional — system-generated notifications (billing events, call completions) have no creator |

**Slack mirroring side-effect**: a `post('save')` hook fires whenever a `target:'specific'` notification is created — `mirrorInAppNotificationToSlack(targetUserId, title, message, type)` (fire-and-forget, errors just logged) posts the same notification to the target user's connected Slack webhook/bot (see §3), if they have one configured. This means **every code path that creates a targeted `Notification`** (billing events, call-completion summaries, workflow actions) automatically gets a Slack echo for free — no per-call-site Slack logic needed.

### API — `/api/notifications` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list — filter is `{target:'all'} OR {target:'specific', targetUserId:me}`; each row enriched with a computed `isRead` (does `readBy` contain my ID); response also includes `unreadCount` |
| GET | `/unread-count` | Just the count, for the navbar badge |
| POST | `/:id/read` | `$addToSet` my ID into `readBy` (idempotent) |
| POST | `/read-all` | Same `$addToSet`, `updateMany` across every notification visible to me that I haven't read yet |

## 2. Account Profile & Security — `/api/settings/account/*` (JWT)

| Method | Path | Purpose |
|---|---|---|
| GET | `/account/profile` | `{name, primaryEmail, avatarUrl, billingAddress, secondaryEmails, connectedAccounts}` — `connectedAccounts` is currently **hardcoded** to a single `{provider:'Google', email: primaryEmail}` entry regardless of actual auth method (there is no real Google OAuth integration in this codebase — this field appears to be a UI placeholder, not a real connected-accounts list; verify before relying on it in a rebuild) |
| PATCH | `/account/profile` | Update `name`/`billingAddress`/`avatarUrl`/`secondaryEmails` (email-validated) |
| POST | `/account/avatar` | Multipart upload (4MB limit) → Cloudinary (`uploadImageBuffer`) → sets `User.avatarUrl`. Throws a clear `ValidationError` if Cloudinary env vars aren't set (no silent base64 fallback here, unlike the agent widget-icon upload) |
| GET | `/account/security` | `{twoFactorEnabled, smsSecurityAlerts}` — `twoFactorEnabled` here is **recomputed** as `settings.twoFactorEnabled && !!settings.totpSecret` (both must be true), not just the raw flag |
| PATCH | `/account/security` | Only accepts `smsSecurityAlerts` — **`twoFactorEnabled` in the body is silently ignored** (destructured out and discarded); 2FA can only be toggled via the dedicated `/account/2fa/setup`→`/verify` and `/account/2fa/disable` endpoints (see `01-auth.md §5.7`), never by a direct PATCH, since enabling it requires the TOTP secret to actually be set up first |
| PATCH | `/account/password` | Calls `changePassword` — requires correct `currentPassword`, same password-complexity rule as registration, and **deletes every `RefreshToken` for the user** on success (changing your password logs you out everywhere, same as the forgot-password reset flow in `01-auth.md`) |
| DELETE | `/account` | Full account + data deletion (§4) |
| GET | `/account/export-data` | GDPR Art. 20 data-portability export (§5) |

(2FA setup/verify/disable and session list/revoke endpoints are documented in `01-auth.md §5.7–5.8` — they're mounted on this same router but belong conceptually to the auth domain.)

## 3. Provider Connections & Integrations

Two parallel but separate systems sharing one underlying model (`ProviderConnection`) and a similar catalog/credentials/connect/disconnect shape — the split is purely a UI/categorization convenience (`SettingsProvidersPage` vs `SettingsIntegrationsPage` on the frontend), not a backend architectural difference.

### `ProviderConnection` (`models/ProviderConnection.model.ts`)
| Field | Type | Notes |
|---|---|---|
| `ownerId` / `providerId` | ObjectId / string | unique compound index `{ownerId,providerId}` — one connection row per user per provider |
| `category` | string | e.g. `'Telephony'`, `'Calendar'`, `'Data Sources'` |
| `credentials` | `Map<string,string>` | raw credential values, **stored in plaintext** in Mongo (not encrypted at rest — see rebuild note) |
| `connected` | boolean | |
| `lastConnectedAt` | Date | |

### Providers catalog (`PROVIDER_CATALOG`, hardcoded in `settings.service.ts`) — category "Telephony/Calendar/Messaging/AI"
- **Twilio** — `account_sid`/`auth_token`/`phone`. On connect, the backend **automates further Twilio setup on the user's behalf**: creates a new Twilio API Key (`client.newKeys.create`) and a TwiML Application pointed at `/api/twilio/widget/voice-twiml` (for browser-widget calling), storing the generated `api_key`/`api_secret`/`twiml_app_sid` back onto the same connection record — this automation is wrapped in try/catch and **failure here doesn't block the connection itself** (logged and continues), since the core account_sid/auth_token/phone are what's actually required for basic calling.
- **Cal.com** — single `api_key` field. Enables the AI booking tool-calls described in `03-calls-realtime-pipeline.md` (Cal.com context resolution).
- **Slack** — supports **either** an Incoming Webhook URL (validated via `isValidSlackWebhookUrl`) **or** a Bot token + channel (validated live against Slack's API via `verifySlackBotToken`) — at least one method is required; if both webhook and bot/channel are left empty, rejected. This is what §1's Slack-mirroring hook posts to.
- A commented-out **Plivo** entry exists in the source (disabled — "until inbound/outbound paths are implemented") — evidence of a partially-planned alternate telephony provider that never shipped.

### Integrations catalog (`INTEGRATION_CATALOG`) — category "Data Sources/CRM & Automation"
- **Google Places** (`available`) — API key for business-lead discovery.
- **Follow Up Boss** (`available`) — CRM sync, API key only — note: connecting this stores credentials but there is **no actual sync logic implemented anywhere in the codebase** found alongside it; this looks like a stubbed/future integration (credentials can be saved, but nothing consumes them yet).
- **Cal.com** (listed a second time under Integrations, separate catalog entry from the Providers-side Cal.com) — same API-key shape.
- **LinkedIn Sales Navigator** (`coming-soon`) and **Advanced CRM Sync** (`disabled`) — placeholder entries, not connectable (`connectIntegration` rejects any status other than `'available'`).

### API — `/api/settings` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/providers` | Merged catalog + connection state; filters `search`/`category`/`status`; credentials returned **masked** (`maskedCredentials` — any key containing `token`/`secret`/`password`/`key` is obscured to `ab**...**yz`-style, first/last chars only) |
| PUT | `/providers/:providerId/connect` | Validates required fields per-provider (with Slack's special either/or rule), runs Twilio automation if applicable, upserts the `ProviderConnection` |
| POST | `/providers/:providerId/disconnect` | Sets `connected:false` (**does not delete the row or clear stored credentials** — a "disconnected" provider retains its saved creds for a quick reconnect) |
| GET | `/integrations` | Same merge pattern for the Integrations catalog |
| PUT | `/integrations/:integrationId/connect` | Same validation pattern |
| POST | `/integrations/:integrationId/disconnect` | Same connect/disconnect model (via the same `ProviderConnection` collection, filtered by category) |
| GET | `/webhook-logs` | Last 50 `Call` docs that have a `webhookSent` field set (i.e. had a post-call webhook attempt) — a simple delivery-log view for the agent `taskSettings.webhookUrl` feature described in `03-calls-realtime-pipeline.md` |
| GET/PATCH | `/notifications` | **Note**: this is a *different* "notifications" than §1's in-app `Notification` feed — this is `UserSettings` fields (`notificationEmails`, `dailySummaryEnabled`, `knowledgeGapAnalysisEnabled`, `defaultCurrency`, `callSuccessThreshold`, `defaultLeadValue`, `aiSuccessScorePrompt`, `secondaryEmails`) — i.e. **email notification preferences and account-level defaults**, not the notification inbox. Worth renaming during a rebuild to avoid the naming collision with the actual `Notification` model/API. |

`GET/PATCH /api/settings/notifications` fields of particular note: `callSuccessThreshold` (0–100, used somewhere in reporting to flag "successful" vs "unsuccessful" calls by score), `defaultLeadValue` (default dollar value assigned to new leads), `aiSuccessScorePrompt` (the custom rubric fed into post-call success evaluation — see `03-calls-realtime-pipeline.md §11`), `knowledgeGapAnalysisEnabled` (gates the knowledge-gap detection feature — see `07-knowledge-base.md §5`).

## 4. Account Deletion (`deleteMyAccount`)

`DELETE /api/settings/account` cascades a hard delete across **every** owned collection in one `Promise.all`: `Agent`, `Call`, `Contact`, `Lead`, `PhoneNumber`, `Campaign`, `CampaignContact`, `Conversation`, `KnowledgeDocument`, `KBChunk`, `Log` (matched by `actorId`, not `ownerId` — a different field name on that model), `RefreshToken`, `Workflow`, `KnowledgeGap`, `UserSettings`, `ProviderConnection`, `ApiKey`, then finally the `User` document itself. This is a genuinely destructive, irreversible, unconfirmed-by-a-second-step operation at the API layer — any confirmation UX (re-enter password, type "DELETE", etc.) lives entirely on the frontend, not enforced server-side.

**Not included in the cascade** (worth double-checking during rebuild if these matter for compliance): `Notification`s targeted at the user, `SecurityAuditLog` rows, and (if billing is kept) `Subscription`/payment history — those rows are left behind referencing a now-deleted `ownerId`/`userId`.

## 5. Data Export (`exportMyData`, GDPR Art. 20)

`GET /api/settings/account/export-data` gathers the user's own `User` (password/2FA fields excluded via `.select()`), `Agent`s, up to 1000 `Call`s (transcripts excluded to keep payload size reasonable), up to 5000 `Contact`s and `Lead`s, `Campaign`s, and (billing-dependent) `Subscription` into one JSON payload, logging a `data_export` security-audit event. This is a straightforward "give the user everything we have on them" endpoint — no special formatting/redaction beyond the two `.select()` exclusions and the row-count caps.

## 6. Rebuild Notes (Replit target, in-house/no-billing)

### Billing coupling
`exportMyData` includes a `Subscription` lookup — remove if billing is stripped. Everything else in this file (Notifications, Provider Connections, Integrations, API Keys, Account Profile/Security, Account Deletion) has **no billing dependency** (aside from `assertApiAccess` gating API-key creation/listing behind a plan feature flag — see `01-auth.md §5.8` — which should likewise be removed).

### Security consideration: plaintext credential storage
`ProviderConnection.credentials` (Twilio auth tokens, Cal.com API keys, Slack bot tokens) are stored as **plaintext strings** in MongoDB — only *masked in API responses*, not encrypted at rest. For an in-house rebuild, especially one storing real customer-facing third-party credentials, consider field-level encryption (e.g. via a KMS-backed envelope encryption scheme, or at minimum `crypto`-based symmetric encryption with a key from env/secrets manager) before this goes to any production use beyond a trusted internal tool.

### Dependencies
`mongoose`, `bcryptjs`, `cloudinary` SDK (avatar upload — swap for S3 per the storage note in `01-auth.md`/`02-agents.md` if Cloudinary is being retired), `twilio` SDK (provider-connect automation), no other third-party deps.

### Decisions to make before rebuilding
- **Follow Up Boss integration**: connectable but has no actual sync implementation anywhere — either implement it or remove the catalog entry to avoid a misleading "connected but does nothing" state.
- **`connectedAccounts` hardcoded Google entry** on the account profile response — either implement real OAuth or remove this fake data before shipping the rebuild's profile page.
- **Rename `/api/settings/notifications`** (email/reporting preferences) to something like `/api/settings/preferences` to stop colliding conceptually with the real `/api/notifications` inbox — purely a clarity improvement, not a behavior change.
- Decide whether Slack-mirroring (§1) is worth keeping for an in-house tool — it's a nice-to-have but adds the Slack provider-connection surface as a dependency of the notification system.
