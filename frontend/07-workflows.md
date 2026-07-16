# 07 — Workflows Pages

> Frontend domain doc. Source: `frontend/src/pages/{WorkflowsPage,WorkflowCanvasPage}.tsx`, `frontend/src/services/workflow.service.ts`. Backend counterpart: `../backend/06-workflows.md`.

## 1. Overview

Two pages mirror the backend's linear `nodes[]` model (`trigger` → `delay`/`call` nodes, see `../backend/06-workflows.md §2`) almost field-for-field: `WorkflowsPage` (`/workflows`) is the list, `WorkflowCanvasPage` (`/workflows/:id`) is the node editor. `workflow.service.ts` is a thin 1:1 wrapper over the backend's 6 endpoints (`list`, `getById`, `create`, `update`, `updateStatus`, `remove`).

## 2. `WorkflowsPage` — list view

Table/card list showing each workflow's name, active/inactive toggle (`Toggle` component calling `workflowService.updateStatus` directly — an optimistic-feeling immediate PATCH, not deferred to a save button), execution count, last-executed time, and a compact **`PipelinePreview`** — a small horizontal strip of colored dots/icons summarizing the node chain (trigger→delay→call) at a glance without opening the full canvas. Clicking a row navigates to `/workflows/:id`.

`CreateModal` here is intentionally minimal — matches the backend's `POST /api/workflows` shape exactly (`{name, agentId?}`), since the real configuration (trigger type, delays, additional call nodes) happens in the canvas editor after creation, not in this modal.

Delete requires a confirm modal (`DeleteWFModal`). A `WarningModal` is shown when the backend rejects an activation attempt (e.g. the validation rules in `../backend/06-workflows.md §3`: no call node configured, scheduled trigger missing phone/time, etc.) — the toggle's error handling surfaces the backend's exact validation message rather than a generic frontend one.

## 3. `WorkflowCanvasPage` — node editor

Despite the "Canvas" name, this is **not a free-form drag-and-drop canvas** — it's a **vertical linear chain** of node cards connected by colored connector lines, matching the backend's strictly-linear `nodes[]` array exactly (no branching, no parallel paths). Node types and their config UI:

### Trigger node (always first, cannot be removed/reordered)
Dropdown of 5 trigger types (`call_completed`, `call_started`, `inbound_call`, `lead_created`, `scheduled`) — labels/descriptions **hardcoded to match the backend's `TRIGGER_LABELS` exactly** (`../backend/06-workflows.md §2`; this duplication between frontend display strings and backend labels is a natural target for a shared constants file in a rebuild). Selecting `scheduled` reveals: a phone number field, a time picker, a frequency selector (`daily`/`weekdays`/`weekly` — with a day-of-week multi-select for `weekly`), and a **timezone dropdown of 12 hardcoded IANA zones** (defaulting to `Asia/Kolkata` — matching the backend's own default, `../backend/06-workflows.md §7`).

### Delay nodes
A dropdown of the same preset options as the backend (`5min` through `72hours`, plus `user_requested` and `custom`) — again hardcoded to mirror `../backend/06-workflows.md §2`'s `DELAY_LABELS`/`PRESET_MINUTES` tables exactly, including the same 7-day (10080-minute) cap on custom delays. The custom-delay UI (`CustomDelayFields`) lets the user pick a unit (minutes/hours/days) and a number, computed back into total minutes client-side before being sent to the backend as `delayCustomMinutes`.

### Call nodes
An agent picker (dropdown of the account's agents) — stores both `agentId` and a denormalized `agentLabel` in the node's `config`, exactly matching the backend's own denormalization pattern (`../backend/06-workflows.md §2`, where the backend re-fills `agentLabel` server-side too as a defensive measure).

### Adding/removing nodes
`addNode('delay'|'call')` appends a new node with sensible default config; `deleteNode(i)` removes by index. There's no "add trigger" option (only one trigger, always first, immutable in type-but-not-config). Each node's display title/subtitle is derived client-side (`deriveDisplay()`) using the **exact same logic** as the backend's `mapWorkflow()`/`ensureNodeShape()` (`../backend/06-workflows.md §2`-adjacent, in `workflow.service.ts`) — meaning this label-derivation logic is currently implemented **twice** (once in the browser for live editing feedback, once on the backend as the source of truth persisted to the DB) — a rebuild could consider deriving labels only server-side and having the frontend just display whatever the backend returns after each save, removing the duplication risk of the two implementations drifting apart.

### Save flow
A single "Save" action submits the whole `nodes` array via `workflowService.update(id, {nodes})` — no per-node save, consistent with the backend's whole-array replacement semantics (`PATCH /api/workflows/:id`, `../backend/06-workflows.md §3`).

## 4. Rebuild Notes

### No billing/out-of-scope-panel coupling
Neither page references `PlanGate`/`useSubscription` or any out-of-scope backend domain — this is a self-contained main-app feature pair.

### Consolidation opportunity
The trigger/delay label tables (`TRIGGER_OPTS`, `DELAY_OPTS`, timezone list, weekday list) are hardcoded **identically** in both `WorkflowCanvasPage.tsx` and the backend's `workflow.service.ts`/`workflowRunner.service.ts`/`workflowScheduler.service.ts` (see `../backend/06-workflows.md §2,8`). A rebuild could define these once (e.g. a shared JSON/constants module, or have the frontend simply fetch a "trigger/delay options" metadata endpoint from the backend) rather than maintaining four independent copies of the same lookup tables across the stack.

### Dependencies
`lucide-react`. No form library. No charting/graph library despite the "Canvas" naming — it's plain flexbox-positioned cards, not an actual canvas element or a node-graph library like React Flow. If a rebuild wants true branching/parallel workflow paths (not just the current linear chain), this would need a real graph-editing library and a corresponding backend schema change (`nodes[]` would need to become a graph, not a flat ordered array) — a substantial change, not a frontend-only tweak.
