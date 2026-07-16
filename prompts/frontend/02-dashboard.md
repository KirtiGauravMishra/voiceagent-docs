# Frontend Prompt 02 — Dashboard Page

> Read `../00-master-rebuild-guide.md`, `01-app-shell-routing-auth.md`, `../backend/03-calls-realtime-pipeline.md` first.

## Goal
`/dashboard` — the post-login landing page: KPI cards, call-volume/status/duration charts, and a recent-calls list.

## Data loading — build a real backend aggregation endpoint

Given this platform's 100k+ calls/day, 1,000+ concurrent target, **do not** fetch hundreds/thousands of raw call rows to the browser and reduce them client-side — that pattern is slow and silently inaccurate the moment call history exceeds whatever page cap you fetch. Instead, add a backend endpoint (e.g. `GET /api/dashboard/stats`) that runs the aggregation in MongoDB and returns already-computed numbers:
- Weekly call volume bucketed by day (last 7 days), split inbound/outbound.
- Status distribution (tally across all calls, or a bounded recent window — decide based on what's useful, not what's cheap).
- Duration-bucket distribution (`0–30s, 30–60s, 1–2m, 2–5m, 5m+`).
- Total/inbound/outbound counts, success rate (% `completed`), average duration, and week-over-week trend deltas (this week vs. the prior week) — all computed with Mongo aggregation (`$group`/`$bucket`/`$facet`), not pulled to Node and reduced there.
- Recent-calls list (top 6 by `createdAt` descending) — a small, cheap, separate query.
- Agent count and campaign count (+ how many campaigns are `running`) — cheap `countDocuments` calls, not full-list fetches.

Have the frontend call this one endpoint on mount and on manual refresh, rather than orchestrating four separate list-service calls and paginating through them in the browser.

## Computed metrics (now server-computed, rendered as-is)
- **Weekly volume** → stacked bar chart (`Sun`..`Sat`), inbound/outbound split.
- **Status distribution** → donut/pie chart, fixed color palette (cycle it if there are more distinct statuses than colors).
- **Duration distribution** → bar chart, the 5 fixed buckets above.
- **KPI cards** → total calls (+ trend), inbound/outbound split, success rate (+ trend), agents/campaigns count (+ running-campaign count).
- **Recent calls** → simple list, status badge + formatted duration (`M:SS`).

## Layout
KPI card row (4 cards) → two chart rows (volume bar chart + status donut; duration bar chart + recent-calls list) → quick-action buttons (New Agent, Phone Numbers, Campaigns, Docs — plain `navigate()` shortcuts). Use **Recharts** (`BarChart`, `PieChart`, `ResponsiveContainer`), theme grid-line/tick colors via your theme context for light/dark contrast (chart fill colors can stay fixed hex values matching your app's accent palette).

Give every card/chart its own skeleton placeholder while loading — don't gate the whole page behind one full-page spinner; let each region resolve independently once its data arrives.

## Dependencies
`recharts`, `lucide-react`.

## Acceptance checklist
- [ ] Dashboard load makes one aggregation request (plus the small recent-calls/counts queries), not a chain of paginated list fetches.
- [ ] KPI numbers and chart data are accurate against a call history in the tens of thousands, not silently capped to a fixed page window.
- [ ] Manual "Refresh" re-triggers the same aggregation call and updates all cards/charts.
- [ ] Light/dark theme toggle keeps every chart legible (grid lines, tick labels, tooltips).
- [ ] Each card/chart shows its own skeleton while loading rather than blocking on a single page-wide spinner.
