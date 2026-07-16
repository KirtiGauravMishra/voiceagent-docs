# 10 — Settings Pages

> Frontend domain doc. Source: `frontend/src/pages/Settings{Notifications,Providers,Developers,Integrations,PhoneNumbers,Profile}Page.tsx`, `frontend/src/services/{settings,phoneNumber}.service.ts`. Backend counterpart: `../backend/08-notifications-settings.md` (Notifications/Providers/Integrations/API-Keys/Profile/Security) and `../backend/02-agents.md`/`../backend/03-calls-realtime-pipeline.md` (phone numbers, referenced from Agent `callSettings.phoneNumberId`).

## 1. `SettingsNotificationsPage` (`/settings/notifications`) — "Alerts" in the sidebar

Despite living under "Notifications," this page actually edits the backend's **email/reporting preferences** object (`UserSettings` fields), not the in-app notification inbox — same naming collision flagged on the backend side (`../backend/08-notifications-settings.md §3`, which recommends renaming this endpoint/page pairing to "Preferences" during a rebuild to avoid confusion with the real Notifications feature, `11-notifications.md`).

Sections: notification email list (add/remove secondary recipient emails), daily-summary toggle, a call-success-score threshold number input (0–100, feeds analytics elsewhere in the app), default lead value, and a **webhook delivery log** table (`getWebhookLogs()` — the last 50 calls that had a `taskSettings.webhookUrl` delivery attempt, see `../backend/03-calls-realtime-pipeline.md §11` and `../backend/08-notifications-settings.md §3`) showing call ID/recipient/status/delivered-or-not.

## 2. `SettingsProvidersPage` (`/settings/providers`) — "Services" in the sidebar

Lists the provider catalog (Twilio, Cal.com, Slack — see `../backend/08-notifications-settings.md §3` for the exact fields/validation per provider) merged with the account's connection state, filterable by search/category/connected-status. Each provider card opens a `ConnectModal` collecting the provider-specific credential fields (masked/password-type inputs for secrets). A dedicated `WebhookUrlBox` component surfaces the Slack incoming-webhook-URL format hint inline. Disconnecting doesn't delete stored credentials server-side (`../backend/08-notifications-settings.md §3`) — the frontend doesn't expose a "forget credentials" action beyond disconnect, matching that backend behavior.

## 3. `SettingsIntegrationsPage` (`/settings/integrations`) — not in the sidebar nav

Reachable only by direct URL (the `NAV_CONFIG`, `01-app-shell-routing-auth.md §6`, has this route commented out as "hidden until needed" but the route itself still works). Same catalog/connect/disconnect pattern as Providers, over a different catalog (Google Places, Follow Up Boss, Cal.com again, LinkedIn Sales Navigator, Advanced CRM Sync — see `../backend/08-notifications-settings.md §3`). Cards for `coming-soon`/`disabled` catalog entries render a status badge instead of a connect button. **Google Places** connecting here is tied to the out-of-scope lead-discovery panel — of the catalog, only Cal.com is directly relevant to the in-scope agent-booking feature (`03-agents.md §2`'s Tools tab).

## 4. `SettingsDevelopersPage` (`/settings/developers`) — "API Keys" in the sidebar

Two halves on one page:
- **API key management** — list/create/revoke keys (`settings.service.listApiKeys/createApiKey/deleteApiKey`, backing `../backend/01-auth.md §5.8`'s `ApiKey` CRUD). `CreateKeyModal` shows the plaintext key **exactly once** on creation (matches the backend's one-time-reveal design).
- **Interactive API reference/playground** (`ApiTester`) — a fully **client-side-hardcoded** `ENDPOINTS` array (method, path, group, params) describing the public API surface, grouped into a sidebar-like nav, with a live "try it" panel that fires real requests against the account's own API key and displays the response. Also includes copyable `CodeSnippet` examples (a fetch-based cURL-equivalent JS snippet). **This endpoint catalog is maintained entirely by hand in this one file** — it will drift out of sync with the real backend API surface (documented per-domain in `../backend/*.md`) unless deliberately kept updated; a rebuild could consider generating this list from the backend's Swagger/OpenAPI spec (`../backend/11-app-bootstrap-infra.md §2` — the backend already runs `swagger-jsdoc`/`swagger-ui-express` at `/api-docs`) rather than hand-maintaining a second copy of the endpoint list in the frontend.

## 5. `SettingsPhoneNumbersPage` — mounted at `/inbound-calls` (primary sidebar item) **and** `/settings/phone-numbers` (legacy redirect)

The most complex Settings page. Manages the `PhoneNumber` collection end-to-end:
- **Buy a number** (`BuyNumberModal`) — country/area-code/type (local vs. toll-free) search against Twilio's live number-search API (`phoneNumberService.search`), then purchase via `phoneNumberService.purchase` (backend provisions it directly through Twilio using the account's connected Twilio credentials, see `../backend/08-notifications-settings.md §3` for how Twilio credentials get connected in the first place).
- **Add an existing number manually** (`AddManualModal`) — for numbers already owned outside this platform's Twilio provisioning flow; requires the user to paste the generated inbound-webhook URL into their own Twilio console's Voice Webhook field themselves (the UI provides a copy-to-clipboard "Copy URL" box, `inboundWebhookUrl(phoneId)` — the exact URL format matching the backend's `/api/twilio/inbound/:phoneNumberId` route, `../backend/03-calls-realtime-pipeline.md §2,9`).
- **Assign inbound agent** — a dropdown per number wiring `PhoneNumber.inboundAgentId`; for Twilio-purchased numbers, assigning an agent typically auto-updates the number's Voice URL in Twilio directly (mentioned in-page copy); manually-added numbers rely on the user having pasted the webhook URL themselves.
- **Set default** — marks one number as the account's default outbound caller ID (`phoneNumberService.setDefault`).
- **Sync with Twilio** (`syncTwilio()`) — reconciles the platform's `PhoneNumber` records against the Twilio account's actual number inventory, reporting created/updated/conflict counts — a bulk "catch up with reality" action, useful if numbers were bought/released directly in the Twilio console rather than through this UI.
- Per-number **"sync webhook"** action re-pushes the correct Voice URL to Twilio for one specific number (a narrower version of the bulk sync, for fixing one out-of-sync number without touching the rest).
- A few in-page help strings are written in **Hinglish** (mixed Hindi/English, e.g. "Step 1: inbound agent select karo...") — worth flagging for a rebuild if the product's target audience/locale strategy differs from the original.

## 6. `SettingsProfilePage` — mounted at both `/settings/profile` and `/settings/account` (two tabs, `?tab=` isn't used — the two routes both render this same component defaulting to different initial tabs based on path, or a shared internal tab state)

- **Profile tab** — name (inline-editable), primary email (read-only display), secondary emails (add/remove list), avatar upload (`uploadAccountAvatar` — Cloudinary-backed on the backend, see `../backend/08-notifications-settings.md §2`), billing address field (labeled "billing address" on the backend but not tied to any actual billing/invoicing flow in the current in-scope feature set — likely a holdover field; confirm relevance during rebuild since billing itself is out of scope).
- **Security tab** — password change form (current+new+confirm, with show/hide toggles), 2FA setup flow inline (start → QR code + manual-entry secret shown → 6-digit confirmation code → enabled; disable requires password + a current TOTP code — exactly mirroring the backend's two-factor-requires-both-factors design, `../backend/01-auth.md §5.7`), and an SMS-security-alerts toggle. Does **not** currently expose the session-list/revoke-session feature documented on the backend (`../backend/01-auth.md §5.8`'s `GET/POST /account/sessions*`) — that backend capability exists but has no corresponding UI in this page; a gap to fill if "view/revoke other logged-in devices" is wanted in the rebuild.

## 7. Rebuild Notes

### Billing-adjacent field to review
`billingAddress` on the Profile tab — confirm whether this is still meaningful with billing out of scope, or should be dropped/repurposed (e.g. as a general company-address field unrelated to payments).

### Out-of-scope catalog entries
`SettingsIntegrationsPage`'s Google Places entry feeds the out-of-scope lead-discovery panel — either drop it from the catalog or keep it inert if Cal.com is the only integration actually consumed by in-scope features.

### Gaps to fill
- **Session management UI** — backend supports list/revoke sessions; no frontend page surfaces it (§6).
- **Hand-maintained API endpoint catalog** (`SettingsDevelopersPage`) will drift from the real backend surface over time — consider deriving it from the backend's OpenAPI spec instead.

### Dependencies
`lucide-react`, `react-hot-toast`. No form library across any Settings page (all plain controlled inputs). `CustomSelect` (shared component, see `12-shared-components-state-styling.md`) is reused across several of these pages.
