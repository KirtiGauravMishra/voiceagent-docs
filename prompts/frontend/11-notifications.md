# Frontend Prompt 11 — Notifications Page

> Read `../00-master-rebuild-guide.md`, `../backend/08-notifications-settings.md`, `01-app-shell-routing-auth.md` first.

## Goal
`/notifications` — the full-page notification center, the real in-app notification inbox (distinct from the Settings "Preferences" page, Prompt `frontend/10-settings.md`, which is an email-preferences page despite its similar name — keep these two clearly separate in naming and content).

## Unread-count tracking

Build one shared source of truth for the unread count, consumed identically by both the Navbar bell badge and this full page — don't let each surface poll independently and risk showing different numbers. Two reasonable approaches, pick one and use it consistently with the rest of the app's state-management choice (per the master guide's "pick one pattern" instruction):
- **React Query**: a single query key (e.g. `['notifications', 'unread-count']`) with a `refetchInterval` of ~30s, consumed by both the Navbar and this page — React Query's cache naturally de-dupes this into one shared value.
- **A small shared hook backed by a module-level store**: one shared counter + subscriber set outside any component, updated by a 30-second poll (only while authenticated) and by explicit local updates (e.g. after marking something read), with a `useNotifications()`-style hook subscribing local state to the shared store on mount.

Either works — just don't reintroduce independent per-component polling for the same number.

## Page structure

Filter chips (`All`/`Unread`/`info`/`warning`/`success`/`alert`), a "Mark all read" action (optimistically update local state), and a paginated list (use the shared `AppPagination` component, Prompt `frontend/12-shared-components-state-styling.md`) where each row shows a type icon/color (the same type→icon/color mapping used in the Navbar dropdown), title, message, and a relative timestamp ("5m ago" style — use one shared date-formatting util, not a copy re-implemented per page).

Clicking an unread row marks it read. Consider deep-linking notifications to their source record (e.g. clicking a "Call Complete" notification navigates to that call's detail page) rather than leaving them purely informational — this is a straightforward improvement worth building from the start rather than retrofitting later.

## Dependencies
`lucide-react`.

## Acceptance checklist
- [ ] The Navbar bell badge and this full page always show the same unread count, without a page refresh needed to reconcile them.
- [ ] Marking a notification read updates both surfaces (Navbar badge and this page) without a manual refresh.
- [ ] "Mark all read" clears the unread count and updates every visible row's read state immediately.
- [ ] Clicking a notification with a linked source record navigates to that record's detail view.
- [ ] Filter chips and pagination both work correctly together (e.g. filtering to "Unread" then paginating doesn't show already-read rows).
