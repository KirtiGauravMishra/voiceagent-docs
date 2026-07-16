# Frontend Prompt 10 — Settings Pages

> Read `../00-master-rebuild-guide.md`, `../backend/08-notifications-settings.md`, `../backend/01-auth.md`, `../backend/03-calls-realtime-pipeline.md`, `../backend/02-agents.md` first.

## Goal
The Settings section: preferences, connected providers, API keys/developer tools, phone-number management, and the profile/security tab — several distinct pages under `/settings/*` plus phone numbers at `/inbound-calls`.

## `SettingsPreferencesPage` (`/settings/notifications` in the sidebar — label it "Preferences" or similar, not "Notifications," to avoid confusion with the actual in-app notification inbox, Prompt `frontend/11-notifications.md`)

Edits email/reporting preferences, not the in-app notification feed. Sections: notification-email list (add/remove secondary recipient emails), daily-summary toggle, a call-success-score threshold number input (0–100), default lead value, and a webhook-delivery log table (recent calls/events that had a webhook delivery attempt — Prompt `backend/10-webhooks.md` — call/event id, recipient, status, delivered-or-not).

## `SettingsProvidersPage` (`/settings/providers` — "Services" in the sidebar)

List the provider catalog (Twilio, Cal.com, Slack) merged with the account's connection state, filterable by search/category/connected-status. Each provider card opens a connect modal collecting provider-specific credential fields (masked/password-type inputs for secrets). Include a small helper component surfacing the Slack incoming-webhook-URL format inline where relevant. Disconnecting should clear the connection status but doesn't need to guarantee credential deletion beyond what the backend already does — match whatever the backend's disconnect endpoint actually does (Prompt `backend/08-notifications-settings.md`).

## `SettingsDevelopersPage` (`/settings/developers` — "API Keys" in the sidebar)

Two halves:
- **API key management** — list/create/revoke keys. The create-key modal shows the plaintext key **exactly once**, at creation time only — never re-displayable after that.
- **Interactive API reference/playground** — group your actual API endpoints into a sidebar-style nav with a live "try it" panel that fires real requests against the account's own API key and displays the response, plus copyable code-snippet examples. **Generate this endpoint list from your backend's OpenAPI/Swagger spec** rather than hand-maintaining a second copy of the endpoint catalog in the frontend — a hardcoded list drifts out of sync with the real API surface the moment either side changes.

## `SettingsPhoneNumbersPage` (mounted at `/inbound-calls`, the primary sidebar item)

The most complex Settings page — manages phone numbers end-to-end:
- **Buy a number** — country/area-code/type (local vs. toll-free) search against your telephony provider's live number-search API, then purchase (the backend provisions it directly using the account's connected provider credentials).
- **Add an existing number manually** — for numbers already owned outside this platform's provisioning flow; the user pastes the generated inbound-webhook URL into their own provider console themselves. Provide a copy-to-clipboard "Copy URL" box with the exact webhook URL for that number.
- **Assign inbound agent** — a dropdown per number wiring the number to an agent; for provider-purchased numbers, assigning an agent should auto-update that number's voice webhook URL with the provider directly; manually-added numbers rely on the user having pasted the webhook URL themselves.
- **Set default** — mark one number as the account's default outbound caller ID.
- **Sync with provider** — a bulk reconciliation action comparing your `PhoneNumber` records against the provider account's actual number inventory, reporting created/updated/conflict counts — useful if numbers were bought/released directly in the provider's console rather than through this UI.
- Per-number **"sync webhook"** action — a narrower version of the bulk sync, for fixing one out-of-sync number without touching the rest.

## `SettingsProfilePage` (`/settings/profile` and `/settings/account`)

- **Profile tab** — name (inline-editable), primary email (read-only), secondary emails (add/remove list), avatar upload (via S3, per the master guide's storage swap).
- **Security tab** — password change form (current+new+confirm, show/hide toggles), 2FA setup flow inline (start → QR code + manual-entry secret → 6-digit confirmation code → enabled; disable requires password + a current TOTP code, mirroring the backend's two-factor-requires-both-factors design), an SMS-security-alerts toggle, and a **session management section** — list active sessions (device/IP/last-active) with a revoke action per session, backed by the backend's session list/revoke endpoints (Prompt `backend/01-auth.md`).

## Dependencies
`lucide-react`, `react-hot-toast`. No form library needed — plain controlled inputs throughout. Reuse the shared `CustomSelect` component (Prompt `frontend/12-shared-components-state-styling.md`) across these pages rather than rolling per-page selects.

## Acceptance checklist
- [ ] Creating an API key shows the plaintext value exactly once; reopening the page never re-displays it.
- [ ] Buying a phone number provisions it end-to-end and inbound calls to it route to the assigned agent without any manual webhook configuration.
- [ ] Manually-added numbers show the correct copyable webhook URL, and calls to that number work once the user pastes it into their provider console.
- [ ] The session list shows real active sessions and revoking one actually invalidates that session's refresh token (verify against the backend, not just UI removal).
- [ ] 2FA setup and disable both work end-to-end, including the QR code scanning and 6-digit verification.
- [ ] The API playground's endpoint list matches your backend's actual current routes (spot-check a few against the OpenAPI spec).
