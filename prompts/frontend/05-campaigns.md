# Frontend Prompt 05 — Campaigns Page

> Read `../00-master-rebuild-guide.md`, `../backend/04-campaigns.md` first.

## Goal
`/campaigns` — list + create wizard + card actions for the outbound-dialing campaign feature. A thin service wrapper over the backend's `/api/campaigns` endpoints (`list`, `getById`, `getContacts`, `create`, `launch`, `pause`, `resume`, `delete`).

## List view & auto-refresh
Load agents + phone numbers (for the create wizard's dropdowns) and a paginated campaign list on mount. **Auto-refresh every few seconds while any campaign is `running`** (same "poll while something is live" pattern as the Calls page) so progress bars/counters update without a manual refresh.

## `CampaignCard`
Name, status pill, assigned agent + phone number, a progress bar (`contactedCount/totalContacts`), and 4 stat tiles (Total/Done/Failed/Skipped, mirroring the backend's `Campaign` counters). Status-conditional action button:
- `draft` → **Launch**
- `running` → **Pause**
- `paused` → **Resume**
- Delete icon shown for any non-`running` status (the backend blocks deleting a running campaign — hide the delete action while running rather than showing it disabled).

Show `scheduledAt` for a scheduled campaign not yet started. Delete opens a confirm modal (name shown, not required to retype — a lighter confirmation than agent deletion).

## `CreateCampaignModal` — two-step wizard

**Step 1 — Setup**: campaign name, agent dropdown (warn if no active agents exist), phone-number dropdown, a "Send Now" vs. "Send Later" toggle (later reveals a `datetime-local` + timezone picker), and dialer settings (`maxConcurrentCalls`, `callIntervalSeconds`, `maxAttempts`, `callWindowStart`/`End`, `timezone` — mirroring the backend's `Campaign.settings` shape).

**Step 2 — Add Contacts**, via three tabs (not merged — only the active tab's selection is used on submit):
- **CSV upload** — drag-drop or file picker, parsed client-side. Provide a "Download Template" link with a sample `name,contact_number,email` CSV.
- **Leads** — fetch leads that have a phone number, searchable multi-select via checkboxes (Prompt `frontend/06-leads-contacts.md`).
- **Contacts** — same pattern against the Contacts directory.

### Scheduled-time timezone conversion (build this exactly — it's easy to get subtly wrong)
When "Send Later" is used, convert the browser's local `datetime-local` value into a UTC ISO timestamp representing the same **wall-clock time in the campaign's selected timezone** (not the browser's own timezone). Do this via a format-then-reparse round trip: render the chosen date into the target timezone as a string, re-parse it, diff against the original value to get the offset, then apply that offset to compute the correct UTC instant. This matters because the backend interprets `callWindowStart`/`callWindowEnd`/scheduling in the campaign's own configured `timezone` field, not server or browser local time — getting this conversion wrong silently schedules campaigns at the wrong wall-clock hour.

## Dependencies
`lucide-react`, `react-hot-toast`. No form library needed — plain controlled inputs. Extract the CSV parsing and the timezone-conversion helper into shared utils if you reuse either elsewhere (e.g. the Contacts CSV import in Prompt `frontend/06-leads-contacts.md` wants similar parsing logic).

## Acceptance checklist
- [ ] Launching a campaign created with "Send Later" at a specific local time in a non-UTC timezone actually starts dialing at the correct wall-clock moment in that timezone.
- [ ] Switching between CSV/Leads/Contacts tabs in Step 2 and submitting only sends the contacts from the currently-active tab.
- [ ] A running campaign's progress bar and stat tiles update live (within a few seconds) without a manual page refresh.
- [ ] Deleting is blocked (button hidden) while a campaign is `running`, and works normally once paused.
- [ ] The CSV template download and a subsequent upload of that same template round-trip correctly.
