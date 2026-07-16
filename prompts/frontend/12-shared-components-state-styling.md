# Frontend Prompt 12 — Shared Components, State Management, Styling & Build Config

> Read `../00-master-rebuild-guide.md` and `01-app-shell-routing-auth.md` first. Every other frontend prompt depends on the primitives and conventions built here — build this prompt's output early, alongside or right after Prompt 01.

## Goal
The shared UI component library, state-management conventions, theming system, design tokens, brand config, and build config every page prompt (02–11) assumes exists.

## No feature-flag/plan-gating layer
This is an in-house tool with no billing/subscription system — every feature is available to every user. Don't build a `PlanGate`-style component or a `useSubscription()` hook at all; there's no upgrade-prompt UI, no per-feature enable/disable check anywhere in the app. Every page prompt in this set assumes this — build features directly, ungated.

## Shared UI primitives (`components/ui/`)

Build these once and **use them everywhere** — the biggest risk on a project this size is each page rolling its own modal/skeleton/badge/pagination markup instead of importing the shared version, which is exactly the inconsistency this prompt exists to avoid.

| Component | Purpose |
|---|---|
| `Modal` | Portal-based modal shell, ESC-to-close, sm/md/lg size variants. Every modal in the app (create/edit/delete/clone confirmations across Agents, Campaigns, Leads, etc.) should be built on top of this, not as an independent `createPortal(...)` call per page. |
| `CustomSelect` | A searchable/portal-positioned dropdown used in place of native `<select>` everywhere (agent config, workflow canvas, voice filters, etc.). |
| `Badge` | Status-pill rendering + one shared set of helper functions mapping call/agent/campaign status strings to color variants — the single source of truth for status colors, used by every page that shows a status pill (Calls, Reports, Campaigns, Leads, Workflows). |
| `AppPagination` | Page-size selector + windowed page-number buttons. Every paginated list in the app (Calls, Campaigns, Reports, Notifications, Leads, Contacts) uses this same component — don't let any page implement its own prev/next pagination inline. |
| `PageLoader` | Branded full-screen/contained loading spinner, theme-aware even before the theme context mounts (reads the pre-hydration theme signal directly, matching the `index.html` script in §3). Used as the top-level `Suspense` fallback. |
| `Spinner` | A small inline spinner for buttons (`disabled={loading}`, swap label for spinner while loading — reuse this pattern on every form-submit button). |
| `SkeletonCard` | Generic pulsing-placeholder card, reused by every page's loading state (Dashboard, Calls, Reports, etc.) instead of a page-local skeleton implementation. |
| `ErrorBoundary` | Class component, catches render errors app-wide (wraps the entire app tree), shows a full-page fallback with a reload button. |

## State management

Per the master guide: **pick one server-state pattern and use it everywhere** — either React Query for every data-fetching hook, or plain `useState`/`useEffect` everywhere. Don't end up with some pages on React Query and most on manual fetching; that's strictly worse than committing to either one consistently.

Real Context usage should stay minimal — this app only needs:
- **Theme** — a React Context (below), the one piece of genuinely cross-cutting UI state.
- **Auth/user** — no Context needed; read directly from `localStorage` via shared auth helpers wherever needed (Prompt `frontend/01-app-shell-routing-auth.md`).
- **Unread notification count** — one shared hook/query (Prompt `frontend/11-notifications.md`), not per-component polling.

## Theming & pre-hydration (`index.html`, `ThemeContext`)

Store the theme in `localStorage`, defaulting to **dark** if unset. Add a synchronous inline `<script>` in `index.html`, running before React mounts, that:
1. Reads the stored theme and applies the `dark`/`light` class + `color-scheme` to `<html>` immediately — zero flash-of-wrong-theme.
2. Sets an inline background/foreground color directly on `<html>` too (not just via a Tailwind class), so there's no visual gap between this script running and your CSS taking over.

Fill the gap until React's first paint with a pure-CSS branded loader (e.g. a pulsing initials mark + shimmer bar) shown while `#root` is empty.

## Design tokens & Tailwind config

- Pick a primary color scale (50–950) and define it in your Tailwind config.
- **Dark-mode surface colors**: define these as CSS custom properties consumed directly in your global stylesheet (e.g. `--c-card`, `--c-surface`, `--c-border`, `--c-muted`), not as hardcoded hex utility pairs (`bg-white dark:bg-[#13141e]`) repeated across dozens of components. If you need Tailwind utilities that reference these variables, use Tailwind's native CSS-variable-based theming (v4) or a small custom plugin — don't let the same literal hex value get typed out in component after component.
- Define a `@layer components` set of reusable classes (`.btn`/`.btn-primary`/`.btn-ghost`/`.btn-danger`/`.btn-outline`, `.card`, `.input`/`.label`/`.field-error`, `.badge*` variants, thin/hidden scrollbar utilities, a few standard keyframe animations) and use these consistently across every page's markup.
- Pick one font family (e.g. Inter) and set it as the default sans stack.

## Brand configuration

One `BRAND` config object (name, initials, tagline, domain, company name/address, contact emails), overridable via `VITE_BRAND_*` env vars, defaulting to **this org's actual identity** — not placeholder/sample branding. Consume it from the sidebar, the loading screen, and any legal/about pages. Make sure the favicon is actually driven by this config (or intentionally kept as a fixed static asset) rather than silently hardcoded separately from the brand initials.

## Build & environment config

- **Dev server**: proxy `/api` to your local backend's dev port; if your dev environment runs inside a container, enable polling-based file watching for reliable hot-reload.
- **Production build**: target a modern JS baseline, minify with `esbuild`, skip sourcemaps in the shipped bundle (or ship them privately if you want error-tracking source mapping), and split vendor chunks sensibly (react, query, forms, ui, http) for better CDN caching with hashed filenames.
- **Env vars** (`VITE_`-prefixed): `VITE_API_URL` (prod backend base URL), `VITE_DEV_PROXY_TARGET` (dev only), the `VITE_BRAND_*` set.
- Gate the production build on type-checking (`tsc && vite build`) — a type error should fail the build, not just warn.

## Dependencies
`tailwindcss`, `postcss`, `autoprefixer`, `vite`, `@vitejs/plugin-react`, `typescript`.

## Acceptance checklist
- [ ] Every modal in the app is built on the shared `Modal` primitive — grep for ad-hoc `createPortal` modal implementations and there should be none outside the primitive itself.
- [ ] Every paginated list uses the shared `AppPagination` component — no page rolls its own prev/next controls.
- [ ] Status badges render identically (same colors for the same status strings) across Calls, Campaigns, Leads, and Workflows.
- [ ] Toggling dark/light mode shows no flash of the wrong theme on a hard page reload.
- [ ] The brand name/logo shown in the sidebar and loading screen reflects this org's actual identity, not placeholder branding.
- [ ] A deliberate TypeScript error in a component fails the production build.
