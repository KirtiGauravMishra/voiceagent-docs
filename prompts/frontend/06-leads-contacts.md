# Frontend Prompt 06 — Leads & Contacts Pages

> Read `../00-master-rebuild-guide.md`, `../backend/05-leads-contacts-crm.md`, `12-shared-components-state-styling.md` first.

## Goal
`LeadsPage` (`/leads` — sales pipeline, list + Kanban) and `DirectoryPage` (`/contacts` — CRM contact directory).

## `LeadsPage` — sales pipeline

List + Kanban dual view over the `Lead` model (7-stage pipeline: `new→contacted→qualified→proposal→negotiation→won/lost`).

### Data loading
Fetch the filtered/paginated lead list and the stage-stats summary **in parallel** on every filter/page change (debounce search ~400ms, reset to page 1 on any filter change). Use the stage-stats endpoint for Kanban column counts.

### View toggle
- **List view** — table with Lead/Contact/Stage/Priority/Value/Source/Created columns, paginated via the shared `AppPagination` component. Show an `emailVerified` badge (valid/risky/catch_all/invalid, color-coded) next to a lead's email if that field is populated — the frontend only displays it, it doesn't trigger verification.
- **Kanban view** — one column per stage, cards show name/company/value/created-date with inline edit/delete icons. Build real **drag-and-drop** stage transitions (dragging a card to a different column updates its `stage`) — this is a standard Kanban-board expectation; don't ship a Kanban view where the only way to move a lead is through an edit-modal dropdown.

### `LeadFormModal`
One form for both create and edit (an `lead` prop present ⇒ edit mode), all `Lead` fields directly editable (name/phone/email/company/value/stage/priority/source/notes), `required` validation on name/phone.

### CSV export
Build this as a real backend-driven export covering the **full filtered result set**, not just the currently-loaded page — a client-only export that silently truncates to one page produces an incomplete file with no warning.

### Summary KPI tiles
Total Leads, Pipeline value, Won count, Conversion % — compute all of these from the **stage-stats endpoint** (which has accurate cross-page totals), not from the in-memory current-page array. Don't mix "accurate cross-page stat" and "current-page-only stat" across different tiles on the same dashboard — pick the accurate source for all of them.

## `DirectoryPage` — CRM contact directory

Standard CRUD table over the `Contact` model: search, `status`/`source` filter dropdowns, tag input on the create/edit form.

### Bulk selection
If you build row multi-select (a `Set` of selected IDs), wire it to an actual bulk action (bulk delete, bulk tag, or export-selected) — don't ship selection checkboxes with no action behind them.

### CSV import/export
- **Import**: a real backend call (`POST /api/contacts/import`) — upsert-by-phone dedup handled server-side (Prompt `backend/05-leads-contacts-crm.md`).
- **Export**: same full-filtered-result-set backend-driven approach as Leads above, not a client-side page-only export.

## Dependencies
`lucide-react`. No form library needed — plain controlled inputs. Reuse the shared `AppPagination` component here (and adopt it on other list pages too — Campaigns/Calls should use the same shared pagination rather than each rolling its own).

## Acceptance checklist
- [ ] Dragging a Kanban card to a different column updates the lead's stage and persists it.
- [ ] Leads summary KPI tiles (pipeline value, conversion %) are accurate across the full dataset, not just the current page, verified against a dataset spanning multiple pages.
- [ ] CSV export on both pages produces a file with every matching row across all pages, not just the loaded page.
- [ ] Re-uploading the same contacts CSV twice does not create duplicate contacts.
- [ ] Any bulk-selection checkbox UI on `DirectoryPage` is connected to a working bulk action.
