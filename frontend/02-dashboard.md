# 02 — Dashboard Page

> Frontend domain doc. Source: `frontend/src/pages/DashboardPage.tsx`.

## 1. Overview

`/dashboard` — the landing page after login. A pure client-side analytics rollup: no dedicated backend "dashboard stats" endpoint exists — the page fetches raw lists from four existing services and computes every KPI/chart client-side with `useMemo`. This is a deliberate architectural choice worth knowing before assuming a backend aggregation endpoint exists to reuse.

## 2. Data loading (`load()`)

Runs once on mount (`useEffect(() => { void load(); }, [])`) and again on manual "Refresh" click. Fetches, **in parallel**:
- `callService.getUserCalls(1, 100)` — first page of up to 100 calls, **then sequentially fetches up to 9 more pages** (`Math.min(firstCalls.totalPages, 10)`, i.e. a hard cap of **1000 calls** pulled client-side) to build the in-browser dataset every chart below is computed from.
- `agentService.getAgents(1, 1)` — page size 1, i.e. this call only exists to read `pagination.total` (agent count) cheaply, not to fetch actual agent data.
- `campaignService.list(1, 100)` — up to 100 campaigns, used both for a total count and to count how many have `status === 'running'`.
- `listConversations(1, 1)` — same "just want the total" pattern as agents.

**Scale consideration**: the "fetch up to 1000 calls, recompute everything client-side on every refresh" pattern is fine at low call volumes but will not hold up at all against the platform's stated 100k-calls/day, 1,000+-concurrent target (see `../backend/03-calls-realtime-pipeline.md §13`) — a user with a long call history will always be capped at the most recent 1000 calls for dashboard purposes (silently — there's no indicator that older calls are excluded from these stats), and the 10 sequential page fetches add real latency to every dashboard load/refresh. At this scale this isn't just a nice-to-have fix — a real backend aggregation endpoint (e.g. `GET /api/dashboard/stats` doing the grouping in MongoDB) is effectively required, since the client-side-reduce-over-1000-rows pattern would give a badly inaccurate (and slow) picture of a dataset this size.

## 3. Computed metrics (`useMemo` blocks, all derived from the same `calls` array)

- **`weeklyData`** — buckets calls from the last 7 days by day-of-week (`Sun`..`Sat`, fixed array regardless of which 7 calendar days are actually in range — so a call from "last Tuesday" and "the Tuesday before" would both land in the same `Tue` bucket if both fall within the 7-day window, which given the 7-day window they can't overlap, but the axis itself doesn't label which actual dates are shown, just day names), split into `inbound`/`outbound` counts, feeding the "Call Volume — Last 7 Days" stacked bar chart.
- **`statusData`** — a `Map` tally of every call's `status` value across the *entire* fetched set (not just last 7 days), rendered as a donut/pie chart with a fixed 6-color palette cycling if there are more than 6 distinct statuses (there are 8 possible values on the backend enum, so this could theoretically cycle once).
- **`durationData`** — 5 fixed buckets (`0–30s, 30–60s, 1–2m, 2–5m, 5m+`) tallying every call's `durationSeconds`.
- **`metrics`** — inbound/outbound totals, `successRate` (% of calls `completed`), `avgDuration`, and **week-over-week trend deltas** for both call volume and success rate (comparing "last 7 days" vs. "the 7 days before that," both computed from the same capped ≤1000-call client-side dataset — so trend accuracy is subject to the same 1000-call ceiling noted above).
- **`recentCalls`** — top 6 by `createdAt` descending, rendered as a simple list with a status badge and formatted duration (`fmtDuration` — `M:SS` format).

## 4. Layout

KPI card row (4 cards: Total Calls w/ trend, Inbound/Outbound split, Success Rate w/ trend, Agents/Campaigns w/ running-campaign count) → two chart rows (call-volume bar chart + status-distribution donut; duration-breakdown bar chart + recent-calls list) → quick-action buttons (New Agent, Phone Numbers, Campaigns, Docs — simple `navigate()` shortcuts, not functional shortcuts themselves). Charts use **Recharts** (`BarChart`, `PieChart`, `ResponsiveContainer`), themed via `useTheme()` to pick appropriately-contrasted grid-line/tick colors for light vs. dark mode (the chart `fill` colors themselves are hardcoded hex values, not theme-derived — same violet/pink/rose palette used elsewhere in the app).

Loading state: every card/chart has its own skeleton placeholder (`Sk` — a generic pulsing gray box component defined inline in this file, not shared) shown while `loading` is true — there's no single full-page spinner, each region degrades to its own skeleton shape.

## 5. Rebuild Notes

### No billing/out-of-scope-panel coupling
This page has no dependency on billing or the other out-of-scope panels — all four data sources it reads (`calls`, `agents`, `campaigns`, `conversations`) are main-app domains. Nothing to strip here.

### Recommended improvement (not a functional bug, but worth doing)
Replace the "fetch up to 1000 calls and reduce client-side" pattern with a real backend aggregation endpoint once the rebuild is under load — see §2. This is the single highest-value change for this page given the platform's stated scale target.

### Dependencies
`recharts`, `lucide-react`. No other services beyond the four already listed (`callService`, `agentService`, `campaignService`, `conversation.service`'s `listConversations`) — all documented in their respective domain files (`04-calls-chats-reports.md`, `03-agents.md`, `05-campaigns.md`).
