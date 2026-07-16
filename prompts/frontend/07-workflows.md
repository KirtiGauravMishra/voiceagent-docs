# Frontend Prompt 07 — Workflows Pages

> Read `../00-master-rebuild-guide.md`, `../backend/06-workflows.md` first.

## Goal
`WorkflowsPage` (`/workflows` — list) and `WorkflowCanvasPage` (`/workflows/:id` — node editor), mirroring the backend's linear `nodes[]` model field-for-field.

## `WorkflowsPage` — list view

Table/card list: name, active/inactive toggle (calls the status-update endpoint directly on toggle — immediate PATCH, not deferred to a save button), execution count, last-executed time, and a compact `PipelinePreview` (a small horizontal strip of colored dots/icons summarizing the node chain — trigger→delay→call — at a glance without opening the full editor). Clicking a row navigates to the canvas page.

Keep the create modal minimal — just `{name, agentId?}` — since real configuration (trigger type, delays, additional call nodes) happens in the canvas editor after creation.

Delete requires a confirm modal. Show a warning modal when the backend rejects an activation attempt (no call node configured, scheduled trigger missing phone/time, etc.) — surface the backend's exact validation message rather than a generic frontend one.

## `WorkflowCanvasPage` — node editor

Not a free-form drag-and-drop canvas — a **vertical linear chain** of node cards connected by connector lines, matching the backend's strictly-linear `nodes[]` array (no branching, no parallel paths).

### Trigger node (always first, cannot be removed/reordered)
Dropdown of 5 trigger types (`call_completed`, `call_started`, `inbound_call`, `lead_created`, `scheduled`) — keep these labels in sync with the backend's own trigger-type labels (ideally sourced from one shared constants definition or a backend metadata endpoint, not duplicated by hand in two places). Selecting `scheduled` reveals: a phone number field, a time picker, a frequency selector (`daily`/`weekdays`/`weekly`, with a day-of-week multi-select for `weekly`), and a timezone dropdown (a reasonable curated list of IANA zones, defaulting to your org's primary timezone).

### Delay nodes
A dropdown of the same preset options as the backend (`5min` through `72hours`, plus `user_requested` and `custom`), with the same 7-day (10080-minute) cap on custom delays. The custom-delay UI lets the user pick a unit (minutes/hours/days) and a number, computed to total minutes client-side before being sent as `delayCustomMinutes`.

### Call nodes
An agent picker (dropdown of the account's agents) — store both `agentId` and a denormalized `agentLabel` in the node's config.

### Adding/removing nodes
Append a new `delay`/`call` node with sensible default config; remove by index. No "add trigger" option — only one trigger, always first. Prefer deriving each node's display title/subtitle from whatever the backend returns after a save, rather than re-implementing the same label-derivation logic independently in the browser — that avoids the two implementations silently drifting apart over time.

### Save flow
One "Save" action submits the whole `nodes` array in a single update call — no per-node save, matching the backend's whole-array-replacement semantics.

## Dependencies
`lucide-react`. No form library, no graph/canvas library needed — plain flexbox-positioned cards are sufficient for the linear-chain UI. (If you ever want true branching/parallel workflow paths instead of a linear chain, that's a real graph-editing library plus a backend schema change from a flat array to a graph — a substantial redesign, not a frontend-only tweak; the linear model is what this prompt builds.)

## Acceptance checklist
- [ ] Toggling a workflow active/inactive immediately reflects the new status and surfaces the backend's validation error clearly if activation is rejected.
- [ ] Adding a trigger → delay → call chain, saving, then reopening the editor shows the exact same chain (round-trips correctly).
- [ ] A `scheduled` trigger's phone/time/frequency/timezone fields all persist and reload correctly.
- [ ] Custom delay values compute the correct total minutes and are capped at 7 days (10080 minutes).
- [ ] The `PipelinePreview` on the list page accurately reflects a workflow's current node chain without needing to open the editor.
