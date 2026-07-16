# 06 — Leads & Directory (Contacts) Pages

> Frontend domain doc. Source: `frontend/src/pages/{LeadsPage,DirectoryPage}.tsx`, `frontend/src/services/{lead,contact}.service.ts`. Backend counterpart: `../backend/05-leads-contacts-crm.md`.

## 1. `LeadsPage` (`/leads`) — sales pipeline

List + Kanban dual view over the tenant-facing `Lead` model (7-stage pipeline: `new→contacted→qualified→proposal→negotiation→won/lost`, see `../backend/05-leads-contacts-crm.md §2`).

### Data loading
`leadService.list({search, stage, priority, page, limit:20})` and `leadService.stageStats()` fetched **in parallel** on every filter/page change (debounced 400ms for search, page reset to 1 on any filter change). `stageStats` powers the Kanban column counts (falls back to counting the currently-loaded page's leads client-side if stats are momentarily unavailable).

### View toggle
- **List view** — a table with Lead/Contact/Stage/Priority/Value/Source/Created columns, paginated via a shared `AppPagination` component (see `12-shared-components-state-styling.md`). Notably renders an **`emailVerified` badge** (valid/risky/catch_all/invalid, color-coded) next to a lead's email — this field is populated by a backend email-verification integration; the frontend here just displays whatever value is already on the lead record, it doesn't trigger verification itself.
- **Kanban view** — one column per stage, cards show name/company/value/created-date with inline edit/delete icons. **Kanban view has no drag-and-drop** — moving a lead between stages requires opening the edit modal and changing the `stage` dropdown; there's no drag handler wired to `KanbanCard`. Worth deciding during rebuild whether drag-and-drop stage transitions are an expected feature (a common Kanban-board expectation) or whether the modal-based edit is acceptable.

### `LeadFormModal`
A single form for both create and edit (`lead` prop present ⇒ edit mode), all `Lead` fields directly editable (name/phone/email/company/value/stage/priority/source/notes) — a flat 1:1 mirror of `CreateLeadInput`, no special validation beyond `required` on name/phone.

### CSV export
Client-side only, exports the **currently-loaded page** of leads (not the full filtered set across all pages) — same limitation pattern as `CallsPage`'s CSV export (`04-calls-chats-reports.md §1`).

### Summary KPI tiles
Total Leads, Pipeline value (`$Xk`, sum of `value` across the **currently-loaded page only** — not a true across-all-pages pipeline total, a subtle accuracy gap worth fixing if pipeline value needs to be accurate at scale), Won count, Conversion % — all computed from the in-memory `leads` array (one page), not from `stageStats` (which *does* have accurate cross-page totals per stage but isn't used for these particular tiles — an inconsistency worth resolving in a rebuild: either compute all summary numbers from `stageStats` for accuracy, or clearly label these tiles as "this page only").

## 2. `DirectoryPage` (`/contacts`) — CRM contact directory

Standard CRUD table over the `Contact` model (`../backend/05-leads-contacts-crm.md §1`): search, `status`/`source` filter dropdowns, bulk row selection (a `Set<string>` of selected IDs — used for... worth checking whether any bulk action actually consumes this selection, e.g. bulk delete/tag; if not currently wired to an action, it's another placeholder-selection-with-no-action pattern like a few others noted elsewhere in this doc set), tag input on the create/edit form, and **CSV import/export**:
- **Import**: `contactService.importCSV(file)` — a real backend call (unlike some other CSV features in this app), hitting `POST /api/contacts/import` (see `../backend/05-leads-contacts-crm.md §1` for the upsert-by-phone dedup behavior).
- **Export**: client-side CSV generation from the loaded page, same pattern/limitation as Leads.

## 3. Rebuild Notes

### No billing/out-of-scope-panel coupling
Both pages are pure main-app CRUD with no `PlanGate`/`useSubscription` usage and no dependency on billing or the other out-of-scope panels.

### Items to verify/decide before rebuild
- **Kanban drag-and-drop** — confirm whether it's an expected feature; currently absent (§1).
- **Bulk-selection actions on `DirectoryPage`** — verify whether the `selected` Set is wired to any actual bulk operation (bulk delete/tag/export-selected) or is dead state; if dead, either implement a bulk action or remove the selection UI.
- **Pipeline-value / summary-tile accuracy** — `LeadsPage`'s KPI tiles are computed from the current page only, not the full filtered/total dataset; decide whether to source these from `stageStats` instead for accuracy.
- **CSV export scope** — both pages export only the loaded page, not the full result set; consider a "export all matching" backend-driven export if full-dataset export is expected (same recommendation as `04-calls-chats-reports.md` and `05-campaigns.md`).

### Dependencies
`lucide-react`. No form library (plain controlled inputs). `AppPagination` (shared component, see `12-shared-components-state-styling.md`) is reused here — a good existing example of the "shared component" pattern other list pages could adopt more consistently (e.g. `CampaignsPage` and `CallsPage` roll their own pagination instead).
