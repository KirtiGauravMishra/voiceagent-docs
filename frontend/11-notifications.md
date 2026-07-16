# 11 — Notifications Page

> Frontend domain doc. Source: `frontend/src/pages/NotificationsPage.tsx`, `frontend/src/services/notification.service.ts`, `frontend/src/hooks/useUnreadNotifications.ts`. Backend counterpart: `../backend/08-notifications-settings.md §1`.

## 1. Overview

`/notifications` — the full-page notification center, essentially a larger version of the `Navbar`'s notification dropdown (`01-app-shell-routing-auth.md §5`) reusing the same `notificationService`. This is the real in-app notification inbox — distinct from `SettingsNotificationsPage` ("Alerts" in the sidebar), which despite its similar name is actually an email-preferences page (see `10-settings.md §1` for that naming-collision note).

## 2. Data & unread-count pub-sub (`useUnreadNotifications`)

Rather than React Query or Context, unread-count tracking is a **hand-rolled module-level pub-sub**: a single `globalCount` variable and a `Set` of subscriber callbacks live outside any component, shared across every mounted consumer. `notifyCountChange(count)` updates the shared value and pushes it to every subscriber; `useUnreadNotifications()` subscribes a local `useState` to that shared store on mount, **and independently polls** `GET /api/notifications/unread-count` every 30 seconds (only while authenticated) to refresh the shared value. Both the `Navbar` badge and this full page call the same hook, so they always show the same count without needing a shared context provider — a lightweight alternative to Context/Redux for this one piece of cross-component state. Worth preserving this pattern (or replacing it with React Query's cache, which would achieve the same de-duplication more idiomatically) rather than reintroducing per-component polling if this page is rebuilt.

## 3. Page structure

Filter chips (`All`/`Unread`/`info`/`warning`/`success`/`alert` — filtering happens **client-side** over the currently-loaded page of notifications, not server-side), a "Mark all read" action (`notificationService.markAllAsRead()`, optimistically updates local state), and a paginated list (20/page) where each row shows a type icon/color (matching the same `type` → icon/color mapping used in the Navbar dropdony, `01-app-shell-routing-auth.md §5`), title, message, and a relative timestamp (`timeAgo` — "5m ago" style, a small local implementation duplicated from similar helpers on `CallsPage`/other pages rather than a single shared date-formatting util).

Clicking an unread row calls `markAsRead(id)` and updates it in place (no navigation — notifications in this app are informational, not deep-linking to the thing they're about, e.g. clicking a "Call Complete" notification doesn't jump to that call's detail page). Confirm during rebuild whether deep-linking notifications to their source record (call/campaign/etc.) is a desired improvement.

## 4. Rebuild Notes

### No billing/out-of-scope-panel coupling
Pure main-app feature. The underlying `Notification` model is written to by both main-app events (call completion, quota warnings — see `../backend/03-calls-realtime-pipeline.md §10`) and the out-of-scope panels' own activity (demo bookings, deal closures, etc., per `../backend/05-leads-contacts-crm.md`) — but the frontend page itself doesn't need to know or care which subsystem created a given notification; it just renders whatever `target:'all'` or `target:'specific'-to-me` rows the backend returns. If the out-of-scope panels are fully removed, this page needs no changes — it will simply stop receiving notifications from those sources.

### Possible improvement
Consider replacing the module-level pub-sub (§2) with React Query's built-in cache/`refetchInterval`, now that React Query is already a dependency — would remove the hand-rolled subscriber-set pattern and unify with the rest of the app's (partial) React Query adoption discussed in `01-app-shell-routing-auth.md §3`.

### Dependencies
`lucide-react`. No pagination component reuse here either (another page rolling its own simple prev/next, similar to `CallsPage`/`CampaignsPage` rather than the shared `AppPagination` used by `LeadsPage`/`DirectoryPage` — see `12-shared-components-state-styling.md` for the full inventory of where `AppPagination` is vs. isn't used).
