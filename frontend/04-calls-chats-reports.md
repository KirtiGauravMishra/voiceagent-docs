# 04 — Calls, Chats & Reports Pages

> Frontend domain doc. Source: `frontend/src/pages/{CallsPage,ChatsPage,ReportsPage}.tsx`, `frontend/src/services/{call,conversation}.service.ts`.

## 1. `CallsPage` (`/calls`) — call log & live monitoring

A three-pane layout: header (KPI summary + refresh/export) → left list (filterable call list) → right detail panel (`CallDetail`, itself split into a left "Details/AI Insights/Overview" tab area and an always-visible right "Conversation" transcript pane).

### Data loading & live refresh
- `callService.getUserCalls(page, 30)` on mount/page-change.
- **Auto-refresh while any call is live**: a `useEffect` watches the loaded `calls` array for any `in-progress`/`ringing` entries and, if found, polls the same endpoint every **4 seconds** (silent — no loading spinner) until nothing is live anymore. The currently-selected call is kept in sync across refreshes (matched by ID, not by array index).
- Search is **debounced 400ms** client-side (filters the already-fetched 30-row page only — this is not a server-side search; searching only searches within the current page of results, a real limitation that needs fixing in the rebuild given the platform's 100k-calls/day scale target).
- **CSV export** (`exportCsv`) is done **entirely client-side** — builds a CSV string from the currently-filtered in-memory rows and triggers a `Blob`+`<a download>` browser download. Only exports what's currently loaded/filtered (one page's worth, 30 rows by default), not the full call history — a limitation to flag if "export all my calls" is an expected feature.

### `CallDetail` panel
- **Left pane** — a call header (direction/status/duration/timestamp, linked contact if any) with an **always-visible recording section**: if `call.recordingUrl` exists, renders a full custom `RecordingPlayer` (play/pause, seekable progress bar computed from `<audio>` element events, download button); if not, shows a "No Recording" placeholder with contextual copy (different message if the call already completed vs. still in progress). The recording audio itself is fetched through the backend's `/api/twilio/recording-proxy` endpoint (see `../backend/03-calls-realtime-pipeline.md §5` and `../backend/10-webhooks.md`) — never a direct Twilio URL, since Twilio recording URLs require Basic-Auth credentials the frontend doesn't have.
  - Three sub-tabs: **AI Insights** (summary/extracted-data/failure-reason, each showing a "not enabled" hint pointing at the relevant Agent → Task setting if empty), **Overview** (quick stat tiles, success-score progress bar, contact card, extracted-field chips, timestamps), **Details** (a flat field/value list including the raw Twilio SID).
  - A **"Mark Complete"** button appears only while a call is live (`in-progress`/`ringing`), opening a confirm dialog that warns "this will bill the call minutes" before calling `POST /api/calls/:id/complete` (see `../backend/03-calls-realtime-pipeline.md §10`) — this is the UI entry point for the backend's manual-completion safety-net endpoint.
- **Right pane** — the conversation transcript, rendered as chat bubbles (agent left-aligned, caller right-aligned) from `transcriptEntries`, falling back to the flat `transcript` string if structured entries aren't available. Auto-scrolls to the newest message on new entries; shows a pulsing "● Live" / "Updating live…" indicator while the call is in-progress.

## 2. `ChatsPage` (`/chats`) — widget conversation history

> **Scope flag**: This page's entire data source (`Conversation` documents, fetched via `listConversations`/`getConversation`) is populated by the **embeddable chat widget** and the agent-detail-page's test-chat panel — both tied to the public widget feature that is out of scope for this documentation pass. The page itself (list + transcript viewer, similar two-pane layout to Calls) is straightforward and could be kept for viewing **test-chat** conversations even if the public widget is dropped, but its practical usefulness shrinks considerably without the widget generating real conversations to review. Flag this page for a scope decision during rebuild rather than assuming it needs full parity treatment.

Brief structure for reference: adapts backend `Conversation` docs (`sessionId`, populated `agentId`, `messages[]`, `source: 'widget'|'test'`, `status: 'active'|'ended'`) into a `ChatItem` display shape; lists conversations with search, shows full message thread on selection (lazy-loads full messages via `getConversation(id)` only if the summary row's message count exceeds what's already loaded); tracks an `online`/offline indicator based on whether the last fetch succeeded.

## 3. `ReportsPage` (`/reports`) — call analytics

Structurally near-identical to `DashboardPage` (`02-dashboard.md`) — **the same "fetch up to 1000 calls across 10 pages, compute everything client-side" pattern** (`callService.getUserCalls` looped up to `Math.min(totalPages, 10)` pages of 100), with the same scale caveat: dashboard/reports accuracy is capped at the most recent ~1000 calls, and this must become a real backend aggregation endpoint given the platform's stated 100k-calls/day, 1,000+-concurrent scale target (see `../backend/03-calls-realtime-pipeline.md §13` and `02-dashboard.md §2`) — at that volume the client-side-reduce approach isn't a minor accuracy gap, it's effectively broken.

Adds, beyond the Dashboard's charts: a **date-range filter** (`7d/30d/90d/all`, computed client-side against `createdAt`), a **status filter dropdown**, a **searchable, paginated data table** (client-side pagination over the filtered in-memory set, `pageButtons` computed to show a windowed page-number strip rather than every page), and inline **recording playback** per table row (`RecordingCell` — same proxy-URL pattern as `CallsPage`'s `RecordingPlayer`, a separate/duplicated implementation rather than a shared component).

## 4. Rebuild Notes

### No billing/out-of-scope-panel coupling in Calls/Reports
`CallsPage` and `ReportsPage` are pure main-app features with no billing dependency.

### `ChatsPage` scope decision
Decide whether to keep this page at all if the public widget is dropped — see the scope flag in §2. If kept only for test-chat review, the "widget" vs. "test" `source` distinction and any widget-specific copy/empty-states should be revisited.

### Recommended improvements (not functional bugs today, but required given the 100k-calls/day target)
- Server-side search/filtering for `CallsPage`'s call list (currently searches only the loaded page).
- A real "export all matching calls" backend endpoint rather than client-side CSV export of just the loaded/filtered rows.
- A real backend analytics/aggregation endpoint for `ReportsPage` (and `DashboardPage`) rather than the capped client-side reduce-over-1000-calls pattern — see `02-dashboard.md §2` for the detailed rationale, which applies identically here.
- Consolidate the duplicated `RecordingPlayer` (`CallsPage`) / `RecordingCell` (`ReportsPage`) components — both wrap the same `/api/twilio/recording-proxy` pattern with near-identical play/pause/download logic.

### Dependencies
`lucide-react`. No charting library on `CallsPage`/`ChatsPage`; `ReportsPage` may reuse patterns from Dashboard but does not appear to use Recharts itself (table + KPI-tile based, not chart-based) — confirm during implementation if a rebuild wants to add charts here too.
