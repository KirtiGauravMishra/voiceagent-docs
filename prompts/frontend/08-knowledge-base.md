# Frontend Prompt 08 — Knowledge Base & Knowledge Gaps Pages

> Read `../00-master-rebuild-guide.md`, `../backend/07-knowledge-base.md` first.

## Goal
`KnowledgeBasePage` (`/knowledge-base` — document upload & management) and `KnowledgeGapsPage` (`/knowledge-gaps` — surfaced answer gaps).

## `KnowledgeBasePage` — document upload & management

A file-manager-style list over the `KnowledgeDocument` collection.

### Upload flow
Drag-and-drop or file picker, restricted client-side to `.pdf`/`.txt`/`.md` (mirroring the backend's upload filter). On submit: multipart upload → the returned document (with `status:'processing'`) is added to the local list immediately (optimistic insert, before the backend finishes processing).

### Status polling
Since processing (extract → chunk → embed → store) is fully asynchronous on the backend, poll: a `setInterval` checks every currently-`processing` entry every few seconds via the status endpoint, updating each entry in place once it flips to `processed`/`failed` (show `errorMsg` inline for failures). Plain polling is fine here — no need for a websocket/SSE push given typical upload frequency.

### List view
Per-document row: file-type icon (color-coded by extension), filename, a copyable `ragId` (the stable public id — distinct from any internal database id), human-formatted file size, status badge, upload date, delete action. Summary stat tiles above the list (processed/processing/failed counts). Search filters by filename client-side (fine to load the full unpaginated document list per account here — this feature's per-account document count stays small compared to, say, calls).

## `KnowledgeGapsPage` — surfaced answer gaps

A worklist over the `KnowledgeGap` analytics: questions the agent couldn't answer well, aggregated across calls.

### Stats row
Total / Pending / Resolved / Ignored / High-severity counts, sourced from the stats endpoint.

### List & filters
`status` filter (pending/resolved/ignored), `sortBy` (recent/severity/oldest — matching the backend's exact sort options), date-range filtering, per-agent breakdown. Each row: question text, a severity pill (color-graded 1–10), hit count, last-seen date, with inline **Resolve**/**Ignore** action buttons — no confirmation modal needed, since a resolved/ignored gap automatically resets to `pending` if it's hit again in a real call (a lighter-weight, easily-reversible action).

### Recompute
A "Recompute" button re-scans recent calls for gaps (useful after first enabling the feature or after tuning detection) — expose the lookback-window (days) as a UI control if you want it configurable, or hardcode a sensible default (e.g. 60 days) and document that choice; either is fine, just make it a deliberate decision.

## Dependencies
`lucide-react`. No charting library needed on either page — stats are numeric tiles, not graphs. Use real pagination on `KnowledgeGapsPage` (this list can grow large); the Knowledge Base document list can stay unpaginated.

## Acceptance checklist
- [ ] Uploading a document shows it as "processing" immediately, then flips to "processed" (with a chunk count) or "failed" (with an error message) within the poll interval, without a manual page refresh.
- [ ] Deleting a document removes it from the list and its underlying chunks are gone (verify via the backend, not just the UI disappearing).
- [ ] Marking a gap "Resolved" removes it from the pending filter view; if the same question is asked again in a new call, it reappears as pending.
- [ ] The severity pill's color scale is visually distinct enough to scan a long list quickly (e.g. green/yellow/red banding).
- [ ] Recompute triggers a visible refresh of the gap list/stats after it completes.
