# 05 — Campaigns Page

> Frontend domain doc. Source: `frontend/src/pages/CampaignsPage.tsx`, `frontend/src/services/campaign.service.ts`.

## 1. Overview

`/campaigns` — a single-file page (list + create wizard + card actions) for the outbound-dialing campaign feature documented on the backend in `../backend/04-campaigns.md`. `campaign.service.ts` is a thin 1:1 wrapper over the backend's `/api/campaigns` endpoints (`list`, `getById`, `getContacts`, `create`, `launch`, `pause`, `resume`, `delete`).

## 2. List view & auto-refresh

Loads agents + phone numbers (for the create wizard's dropdowns) and campaigns (paginated, 20/page) on mount. **Auto-refreshes every few seconds while any campaign is `running`** (mirrors the same "poll while something is live" pattern used on `CallsPage`/`ChatsPage`) so progress bars/counters update without a manual refresh. Each campaign renders as a `CampaignCard`.

## 3. `CampaignCard`

Shows name, status pill, assigned agent + phone number, a **progress bar** (`contactedCount/totalContacts`), and 4 stat tiles (Total/Done/Failed/Skipped — direct mirror of the backend's `Campaign` counters, see `../backend/04-campaigns.md §2`). Status-conditional action button:
- `draft` → **Launch**
- `running` → **Pause**
- `paused` → **Resume**
- Delete icon shown for any non-`running` status (backend blocks deleting a running campaign — see `../backend/04-campaigns.md §3` — the frontend mirrors that by simply hiding the delete button while running, rather than showing it disabled).

A scheduled campaign not yet started shows its `scheduledAt` time. Delete requires a confirm modal (`DeleteCampaignModal`, name shown but not required to be retyped — a lighter confirmation than the Agent deletion flow in `03-agents.md`).

## 4. `CreateCampaignModal` — two-step wizard

**Step 1 — Setup**: campaign name, agent dropdown (warns if no active agents exist), phone-number dropdown, a "Send Now" vs. "Send Later" toggle (later reveals a `datetime-local` + timezone picker), and the dialer settings (`maxConcurrentCalls`, `callIntervalSeconds`, `maxAttempts`, `callWindowStart`/`End`, `timezone` — direct mirrors of the backend's `Campaign.settings` shape, see `../backend/04-campaigns.md §2`).

**Step 2 — Add Contacts**, via **three interchangeable sources** (tabs, not merged — only the active tab's selection is used):
- **CSV upload** — drag-drop or file-picker, parsed client-side (`parseCSVToContacts`, a local CSV parser — not a shared util with the backend's CSV import). A "Download Template" link produces a sample `name,contact_number,email` CSV.
- **Leads** — fetches up to 100 leads (`leadService.list`, filtered to those with a `phone`), searchable, multi-select via checkboxes (see `06-leads-contacts.md` for the Leads feature itself).
- **Contacts** — same pattern against the Contacts directory (`contactService.list`).

Whichever tab is active when the user submits determines `finalContacts` — switching tabs does not merge selections across sources.

### Scheduled-time timezone conversion
When "Send Later" is used, the browser's local `datetime-local` value is converted into a UTC ISO timestamp that represents the same **wall-clock time in the campaign's selected timezone** (not the browser's own timezone) — done via a `toLocaleString` round-trip trick (format the date into the target timezone as a string, re-parse it, diff against the original to get the offset, apply that offset to get the correct UTC instant). This matters because the backend interprets `callWindowStart`/`callWindowEnd`/scheduling in the campaign's configured `timezone` field (`../backend/04-campaigns.md §6`), not the server's or the browser's local time — a subtle but important piece of logic to preserve exactly if this wizard is rebuilt, since getting it wrong would silently schedule campaigns at the wrong wall-clock time.

## 5. Rebuild Notes

### No billing coupling directly in this page
Unlike Agents/Calls, `CampaignsPage` itself doesn't reference `useSubscription`/`PlanGate` — the plan-tier concurrency cap is enforced entirely server-side (`../backend/04-campaigns.md §9`, itself flagged there as needing a rebuild decision). If that backend gate is removed/changed, no frontend change is needed here since the frontend never displayed or gated on that limit directly (contrast with `AgentDetailPage`'s Call tab, which does show a read-only "Max Concurrent Calls" plan-derived number — see `03-agents.md §2`).

### Dependencies
`lucide-react`, `react-hot-toast`. No form library (plain controlled inputs, consistent with Agents/Calls pages). The CSV parsing (`parseCSVToContacts`) and the timezone-conversion trick (§4) are both local to this file — worth extracting to shared utils if either is reused elsewhere in a rebuild (e.g. the Directory/Contacts CSV import, which likely wants similar parsing logic).
