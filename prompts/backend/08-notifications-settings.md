# Backend Prompt 08 — Notifications, Settings & Provider Connections

> Read `../00-master-rebuild-guide.md`, `01-auth.md` first. Read `03-calls-realtime-pipeline.md` too if you're building this before that one — the `ProviderConnection` model here is what the Cal.com appointment-booking flow in Prompt 03 looks up.

## Goal
Three domains: in-app `Notification` records (a bell-icon feed), per-user `UserSettings` (profile/preferences not already covered by auth — see Prompt 01 for the auth-owned `UserSettings` fields like 2FA), and `ProviderConnection` (per-owner credentials for third-party integrations — Twilio, Cal.com, Slack — that other modules depend on being connected).

## Data model — `Notification`

`ownerId`, `type` (enum — e.g. `call_completed|campaign_completed|workflow_executed|low_balance_removed_n_a|system|lead_created` — tailor the enum to whichever events this rebuild actually emits; drop any billing-flavored types since there's no billing system here), `title`, `message`, `read` (bool, default `false`), `link` (optional in-app path the notification should navigate to when clicked), `metadata` (loose object — e.g. `{callId}`/`{campaignId}`).

Index `{ownerId, read, createdAt:-1}` (powers "unread count" and the paginated feed).

**Creation**: notifications are created server-side by other modules (call completion, campaign completion, workflow execution, etc.) via a single shared `createNotification(ownerId, type, title, message, opts)` helper — don't scatter ad-hoc `Notification.create()` calls across the codebase; funnel them through one function so delivery (see below) stays consistent.

**Delivery**: at minimum, persist to Mongo for the in-app feed. If you want live push, emit over the existing WebSocket layer (reuse the Media Streams socket infra's connection-management patterns from Prompt 03, or a lightweight separate Socket.IO/`ws` namespace scoped by `ownerId`) so an open dashboard tab updates without polling. Polling (e.g. every 30–60s) is an acceptable fallback if you don't want to stand up a second realtime channel.

## API — `/api/notifications` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | paginated, filter `read`/`type` |
| GET | `/unread-count` | single number, for the bell badge |
| PATCH | `/:id/read` | mark one read |
| PATCH | `/read-all` | mark all read for this user |
| DELETE | `/:id` | delete one |
| DELETE | `/` | clear all |

## Data model — `UserSettings` (general preferences)

Distinct from the auth-security settings in Prompt 01 (2FA etc.) — this covers product preferences: `ownerId`, `timezone` (default `'UTC'`), `dateFormat`, `notificationPreferences` (embedded bools per channel/type — e.g. `{emailOnCallCompleted, emailOnCampaignCompleted, inAppEnabled}`), `theme` (`light|dark|system`, default `system` — though theme is often purely client-side `localStorage`; only persist server-side if you want it to follow the user across devices).

If Prompt 01 already created a single `UserSettings` collection for 2FA/security, **extend that same document** with these fields rather than creating a second collection — one settings document per user, not two.

## API — `/api/settings` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | get current user's settings (create with defaults on first read if missing) |
| PATCH | `/` | partial update, same merge-not-overwrite discipline as Prompt 02's agent sub-documents |

## Data model — `ProviderConnection` (Twilio / Cal.com / Slack)

`ownerId`, `provider` (enum `twilio|calcom|slack`), `status` (enum `connected|disconnected`), `credentials` (**encrypted at rest** — see below), `metadata` (loose, provider-specific display info — e.g. for Cal.com, the connected account's email/username so the UI can show "Connected as ..."), `connectedAt`.

Unique compound index `{ownerId, provider}` — one connection row per provider per owner.

**Credential encryption**: never store API keys/tokens in plaintext. Encrypt `credentials` with an app-level symmetric key (`ENCRYPTION_KEY` env var, AES-256-GCM or similar) before writing to Mongo, and decrypt only in-memory at the point of use (placing a Twilio call, calling the Cal.com API). Never return decrypted credentials in any API response — the `GET` endpoint below returns only `status`/`metadata`, never `credentials`.

**Per-provider credential shape**:
- `twilio`: `{accountSid, authToken}` — used to place calls/purchase numbers under the owner's own Twilio account instead of (or in addition to) a platform-level default.
- `calcom`: `{apiKey}` (Cal.com API keys are the simplest integration path; use OAuth instead if you want a "Connect with Cal.com" button rather than an API-key paste field — either is fine, just be consistent with what the frontend prompt describes) plus `metadata:{username}` for display.
- `slack`: `{webhookUrl}` (an incoming webhook URL is enough for simple notification delivery; use a full OAuth app if richer Slack integration is wanted).

## API — `/api/settings/providers` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | list all providers with connection status + metadata (never raw credentials) |
| POST | `/:provider/connect` | body = provider-specific credential fields; validate them with a real test call to that provider (e.g. a lightweight Cal.com API ping, a Twilio account-fetch) before saving — don't save unverified credentials as "connected" |
| POST | `/:provider/disconnect` | clear `status` to `disconnected`; deleting the encrypted credentials entirely is also acceptable, but at minimum the connection must stop being usable |

**Resolving a connection at call time** (used by Prompt 03's appointment-booking flow, and by campaign/number-purchase flows that want the owner's own Twilio account): a single `getProviderConnection(ownerId, provider)` helper that loads + decrypts credentials, returns `null` if not connected, and is the **only** place decryption happens — don't scatter decryption calls across modules.

## Environment variables
`ENCRYPTION_KEY` (for `ProviderConnection.credentials` encryption — required), plus whatever earlier prompts already define (email provider creds, if `notificationPreferences` gates actual email sends).

## Acceptance checklist
- [ ] Completing a call/campaign/workflow creates a `Notification` visible in `GET /api/notifications` without any manual trigger.
- [ ] `unread-count` reflects reality after individually marking items read and after `read-all`.
- [ ] `PATCH /api/settings` with a partial `notificationPreferences` object only changes the given sub-fields.
- [ ] A brand-new user's first `GET /api/settings` returns sensible defaults rather than a 404.
- [ ] Connecting Cal.com with an invalid API key is rejected at connect-time (a real verification call fails), not silently saved as "connected."
- [ ] `GET /api/settings/providers` never returns a raw credential value, even to the owning user.
- [ ] Disconnecting a provider actually stops it from being usable (e.g. Prompt 03's booking flow correctly treats it as not-connected afterward).
