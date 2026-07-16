# Frontend Prompt 04 — Calls, Chats & Reports Pages

> Read `../00-master-rebuild-guide.md`, `../backend/03-calls-realtime-pipeline.md`, `../backend/10-webhooks.md`, `02-dashboard.md` first.

## Goal
`CallsPage` (`/calls` — call log + live monitoring), `ChatsPage` (`/chats` — text-chat conversation history, if you build a test-chat feature for agents), `ReportsPage` (`/reports` — call analytics table + charts).

## `CallsPage` — call log & live monitoring

Three-pane layout: header (KPI summary + refresh/export) → left list (filterable call list) → right detail panel (`CallDetail`, split into a left tab area — Details/AI Insights/Overview — and an always-visible right transcript pane).

### Data loading & live refresh
- Fetch a page of recent calls on mount/page-change.
- **Auto-refresh while any call is live**: watch the loaded calls for any `in-progress`/`ringing` entries; if found, silently poll every ~4 seconds (no loading spinner) until nothing is live. Keep the currently-selected call in sync across refreshes, matched by ID not array index.
- **Search should hit the backend**, not just filter the currently-loaded page — at this platform's call volume, client-side search over one page of results would miss the vast majority of a user's history. Debounce ~400ms before firing the server request.
- **CSV export should be a real backend endpoint** that streams/exports matching rows server-side (respecting the active filters), not a client-side `Blob` built from whatever happens to be loaded in memory — a client-only export silently caps at one page and gives users an incomplete file with no indication anything was left out.

### `CallDetail` panel
- **Left pane** — call header (direction/status/duration/timestamp, linked contact if any) with an always-visible recording section: if a recording exists, a full custom `RecordingPlayer` (play/pause, seekable progress bar driven by `<audio>` element events, download button); if not, a "No Recording" placeholder with contextual copy (different message for completed vs. still-in-progress calls). Fetch recording audio through your backend's recording-proxy endpoint (Prompt `backend/03-calls-realtime-pipeline.md`) — never a direct provider URL, since telephony-provider recording URLs typically require server-side credentials the frontend doesn't have.
  - Three sub-tabs: **AI Insights** (summary/extracted-data/failure-reason, each showing a "not enabled" hint pointing at the relevant Agent → Task setting if empty), **Overview** (quick stat tiles, success-score progress bar, contact card, extracted-field chips, timestamps), **Details** (a flat field/value list including the raw provider call SID).
  - A **"Mark Complete"** button appears only while a call is live, opening a confirm dialog before calling the manual-completion endpoint (Prompt `backend/03-calls-realtime-pipeline.md`) — the UI entry point for the backend's stuck-call safety-net.
- **Right pane** — conversation transcript as chat bubbles (agent left-aligned, caller right-aligned), falling back to a flat transcript string if structured entries aren't available. Auto-scroll to the newest message; show a pulsing "● Live" indicator while the call is in-progress.

## `ChatsPage` — text-conversation history

Build this only if you're keeping a test-chat feature on the agent detail page (a way to try an agent via text instead of a live call) — it has no dependency on any public-facing feature. List conversations (search by content/agent), show the full message thread on selection, lazy-load full messages if the summary row's message count exceeds what's already loaded. Two-pane layout matching `CallsPage`'s structure.

## `ReportsPage` — call analytics

Same KPI/chart set as `DashboardPage` (Prompt `frontend/02-dashboard.md`), backed by the same real backend aggregation endpoint — do not re-implement a second client-side reduce-over-many-calls pattern here. Adds beyond the dashboard:
- A date-range filter (`7d/30d/90d/all`), passed to the backend aggregation as a query param, not applied client-side.
- A status filter dropdown, same treatment.
- A searchable, **server-paginated** data table (windowed page-number strip, not client-side pagination over an in-memory set).
- Inline recording playback per row — reuse the same `RecordingPlayer` component from `CallsPage` rather than a second duplicated implementation.

## Dependencies
`lucide-react`. `ReportsPage` can reuse Recharts from the Dashboard prompt if you want charts there too, beyond the data table.

## Acceptance checklist
- [ ] Searching calls returns results across the user's full call history, not just the currently-loaded page.
- [ ] CSV export produces a file containing every matching row, not just the currently-loaded page.
- [ ] A live call's status/transcript updates in the UI within ~4 seconds of a real status change, without a visible loading flicker.
- [ ] The recording player works for a completed call with a recording and shows the correct empty state for one without.
- [ ] `ReportsPage`'s date-range and status filters change the actual data returned (server-side), not just what's displayed from an already-fetched set.
- [ ] `RecordingPlayer` is a single shared component used by both `CallsPage` and `ReportsPage`.
