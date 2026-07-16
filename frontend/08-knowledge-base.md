# 08 — Knowledge Base & Knowledge Gaps Pages

> Frontend domain doc. Source: `frontend/src/pages/{KnowledgeBasePage,KnowledgeGapsPage}.tsx`, `frontend/src/services/{knowledge,knowledgeGap}.service.ts`. Backend counterpart: `../backend/07-knowledge-base.md`.

## 1. `KnowledgeBasePage` (`/knowledge-base`) — document upload & management

A file-manager-style list over the backend's `KnowledgeDocument` collection (`../backend/07-knowledge-base.md §2`).

### Upload flow (`UploadModal`)
Drag-and-drop or file-picker, restricted client-side to `.pdf`/`.txt`/`.md` (mirroring the backend's `multer` `fileFilter`, `../backend/07-knowledge-base.md §3`). On submit: `knowledge.service.uploadDocument(file)` → multipart POST → the returned `{id, ragId}` is immediately added to the local list with `status:'processing'` (optimistic insert, before the backend has actually finished processing).

### Status polling
Since document processing (`extract → chunk → embed → store`) is fully asynchronous on the backend (`../backend/07-knowledge-base.md §3`), the page runs a **client-side poll**: a `setInterval` checks all currently-`processing` entries every few seconds via `getDocumentStatus(id)`, updating each entry's status in place once it flips to `processed`/`failed` (an `errorMsg` is shown inline for failures). This is a simple polling loop, not a websocket/SSE push — acceptable for the expected upload frequency of this feature, but worth noting as a design choice if a rebuild considers real-time push instead.

### List view
Per-document row: file-type icon (color-coded by extension), filename, a copyable `ragId` (the stable public UUID — see `../backend/07-knowledge-base.md §2` for why this differs from the Mongo `_id`), file size (human-formatted), status badge, upload date, delete action. Summary counts (processed/processing/failed) shown as small stat tiles above the list. Search filters by filename client-side only (all documents for the account are loaded in one list call — no pagination on this page, consistent with the backend's `listDocuments` returning the full unpaginated set per user, `../backend/07-knowledge-base.md §3`).

## 2. `KnowledgeGapsPage` (`/knowledge-gaps`) — surfaced answer gaps

A dashboard over the backend's `KnowledgeGap` analytics (`../backend/07-knowledge-base.md §5`): questions the agent couldn't answer well, aggregated across calls.

### Stats row
Total / Pending / Resolved / Ignored / High-severity counts, sourced from `knowledgeGapService.stats()` (`../backend/07-knowledge-base.md §6`'s `GET /api/knowledge-gaps/stats`).

### List & filters
`status` filter (pending/resolved/ignored), `sortBy` (recent/severity/oldest — matching the backend's exact sort options), date-range filtering, and per-agent breakdown. Each row shows the question text, a **severity pill** (color-graded 1–10, matching the backend's severity computation described in `../backend/07-knowledge-base.md §5`), hit count, and last-seen date, with inline **Resolve**/**Ignore** action buttons calling `knowledgeGapService.updateStatus(id, status)` directly (no confirmation modal — a lighter-weight action than most other status-changing UI in this app, appropriate since it's easily reversible by re-triggering the same question in a future call, per the backend's own "resets to pending if hit again" behavior noted in `../backend/07-knowledge-base.md §5`).

### Recompute
A "Recompute" button calls `knowledgeGapService.recompute(60)` (fixed 60-day lookback in this page — not user-configurable in the UI despite the backend endpoint accepting any `days` value 1–180, `../backend/07-knowledge-base.md §5`) to re-scan recent calls for gaps, useful after first enabling the feature or after tuning detection.

## 3. Rebuild Notes

### No billing/out-of-scope-panel coupling
Both pages are pure main-app features — no `PlanGate`/`useSubscription`, no dependency on billing or the other out-of-scope panels. (The Knowledge Base *attachment* picker inside `AgentDetailPage`'s LLM tab *is* wrapped in `PlanGate feature="knowledgeBase"` — see `03-agents.md §2` — but these two standalone pages themselves are not gated.)

### Minor UX inconsistency to consider fixing
The Recompute lookback window is hardcoded to 60 days in the frontend despite the backend supporting a configurable 1–180 day range — either expose this as a UI control or leave as-is with a documented rationale (the current 60-day default is a reasonable middle ground, so this may be a non-issue — just worth a conscious decision rather than an oversight).

### Dependencies
`lucide-react`. No pagination component on `KnowledgeBasePage` (single unpaginated list) vs. real pagination on `KnowledgeGapsPage` (matches the backend's paginated `/knowledge-gaps` endpoint). No charting library on either page — all stats are numeric tiles, not graphs.
