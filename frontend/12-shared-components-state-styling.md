# 12 — Shared Components, State Management, Styling & Build Config

> Frontend domain doc. Source: `frontend/src/components/{layout,ui,agents,calls}/*`, `frontend/src/hooks/*`, `frontend/src/contexts/ThemeContext.tsx`, `frontend/src/config/{brand,navigation}.ts`, `frontend/tailwind.config.js`, `frontend/vite.config.ts`, `frontend/index.html`, `frontend/index.css`.

## 1. Layout components

`AppLayout`, `Sidebar`, `Navbar` are covered in depth in `01-app-shell-routing-auth.md §5` — not repeated here. Summary for cross-reference: `AppLayout` is the `Private`-route shell (fixed sidebar+navbar, collapsible); `Sidebar` is driven entirely by `config/navigation.ts`'s `NAV_CONFIG`; `Navbar` hosts the page title, global search modal, theme toggle, notifications dropdown, and profile menu.

## 2. `PlanGate` — the billing-feature-flag gate

The single most-reused billing-coupled component in the app. `<PlanGate feature="...">` checks `useSubscription().subscription?.featuresEnabled[feature]` (defaulting to `true` if the field is missing — i.e. **fails open**, not closed, if the subscription payload doesn't include a given feature key) and renders `children` if enabled, else one of three fallback presentations:
- `variant="full"` (default) — a full-width "Upgrade Your Plan" card with a lock icon and a link to `/billing`.
- `variant="button"` — dims the wrapped content and overlays an "Upgrade" pill on hover.
- `variant="inline"` — a small inline "🔒 Upgrade required" badge.

Known feature keys in use across the app: `inboundCalls`, `outboundCalls`, `voiceWidget`, `chatWidget`, `automationFlow`, `knowledgeBase`, `training`, `autoPromptTemplates`, `elevenLabsVoice` (checked directly via `subscription?.featuresEnabled?.elevenLabsVoice` in `AgentDetailPage` rather than through `PlanGate` itself — see `03-agents.md §2`). Used across `AgentsPage`/`AgentDetailPage` (`03-agents.md`) most heavily; not used at all on Campaigns, Leads/Contacts, Workflows, Voices, or Notifications pages.

### `useSubscription()` hook
A plain (non-React-Query) hook: fetches `billingService.getSubscription()` once on mount, exposes `{subscription, loading, error, isFeatureEnabled(feature)}`. **This is the hard dependency point on the billing backend** for the entire feature-gating system — `billingService.getSubscription()` calls the backend's billing domain (out of scope, see `../backend/`'s billing exclusion). For an in-house/no-billing rebuild, this hook (and by extension `PlanGate`) needs a decision: either stub `getSubscription()` to always return every feature enabled (simplest — keeps the `PlanGate` component shape intact but makes it a no-op), or remove `PlanGate`/`useSubscription` entirely and unwrap every `<PlanGate feature="...">{children}</PlanGate>` back down to just `{children}` across every page that uses it (cleaner long-term, more surface area to touch). Given how many pages reference this component (`03-agents.md`, and potentially others not yet flagged), the stub-it-out approach is the lower-risk path for an initial rebuild pass.

## 3. Shared UI primitives (`components/ui/`)

| Component | Purpose |
|---|---|
| `Modal.tsx` | Portal-based modal shell, ESC-to-close, sm/md/lg size variants. Notably **not** used by most of the app's own modals — `AgentsPage`, `CampaignsPage`, `LeadsPage`, etc. each roll their own `createPortal(...)` modal markup inline rather than wrapping this shared component (see the "duplicated modal" notes in `03-agents.md §5`). A rebuild could consolidate every page-local modal onto this one shared primitive. |
| `Select.tsx` (`CustomSelect`) | A custom searchable/portal-positioned dropdown used in place of native `<select>` almost everywhere (Agent config, Workflow canvas, Voices filters, etc.) — one of the more consistently-reused primitives in the app. |
| `Badge.tsx` | Status-pill rendering + helper functions mapping call/agent status strings to color variants — though several pages (Calls, Reports, Campaigns) define their **own** local `STATUS_CFG`-style maps instead of using this shared helper, so status-color consistency is enforced by convention/copy-paste rather than a single source of truth. |
| `AppPagination.tsx` | Page-size selector + windowed page-number buttons. Used by `LeadsPage`/`DirectoryPage`; **not** used by `CallsPage`, `CampaignsPage`, `ReportsPage`, or `NotificationsPage`, each of which implements its own simpler prev/next or page-number pagination inline. Consolidating onto this one component is a recurring "worth doing in a rebuild" theme across several of these docs. |
| `PageLoader.tsx` | Branded full-screen/contained loading spinner, theme-aware even before React's own theme context mounts (reads the pre-hydration theme signal directly, matching the `index.html` script described in §5). Used as the `<Suspense>` fallback in `App.tsx` (`01-app-shell-routing-auth.md §2`). |
| `Spinner.tsx` | A plain small spinner, used inline in buttons (`disabled={loading} ... {loading ? <Spinner/> : 'Label'}` — a pattern repeated across nearly every form-submit button in the app). |
| `SkeletonCard.tsx` | Generic pulsing-placeholder card; several pages (Dashboard, Calls) instead define their own local `Sk`/skeleton components rather than using this shared one — same duplication pattern as Badge/Pagination. |
| `ErrorBoundary.tsx` | Class component, catches render errors app-wide (wraps the entire `<App/>` tree, `01-app-shell-routing-auth.md §2`), shows a full-page fallback with a reload button. |

**Overall theme across this component inventory**: the app has a real, usable set of shared UI primitives, but adoption is **inconsistent** — many pages duplicate modal/skeleton/status-badge/pagination logic locally instead of importing the shared version. This is a recurring, cross-cutting rebuild opportunity (also called out per-page in `03-agents.md`, `10-settings.md`, `11-notifications.md`) rather than a single fix — worth a dedicated consolidation pass if the rebuild wants to reduce duplicated UI code.

## 4. State management summary (cross-cutting view)

This ties together observations made per-page elsewhere in this doc set:
- **React Query**: configured globally (`01-app-shell-routing-auth.md §3`) but only actually used by `hooks/useAgents.ts` (`03-agents.md`) and `hooks/useCalls.ts`. Every other page manages server state with plain `useState`/`useEffect`/manual try-catch fetches. A rebuild should pick one direction — full adoption or full removal — rather than carrying the partial state forward.
- **`useAgents(page)`** additionally layers a manual `sessionStorage` cache (`va_agents_cache_p{page}`) on top of React Query specifically to avoid a skeleton flash on repeat visits — a targeted UX fix, not a general pattern used elsewhere.
- **`useSubscription()`** — plain fetch-on-mount hook, the billing-feature-flag data source for `PlanGate` (§2).
- **`useUnreadNotifications()`** — a bespoke module-level pub-sub + 30s poll, detailed in `11-notifications.md §2`.
- **Theme** — React Context (`ThemeContext`, next section), the only "real" Context in the app.
- **Auth/user** — no Context at all; read directly from `localStorage` via `utils/auth.ts` helpers wherever needed (`01-app-shell-routing-auth.md §4`).

## 5. Theming & pre-hydration (`index.html`, `contexts/ThemeContext.tsx`)

`localStorage['vx_theme']` (falling back to a legacy `va_theme` key for backwards compatibility, defaulting to **dark** if neither is set) drives the theme. A synchronous inline `<script>` in `index.html` — running **before** React mounts — reads this key and:
1. Applies the `dark`/`light` class and `color-scheme` to `<html>` immediately (zero flash-of-wrong-theme).
2. Sets an inline background/foreground color directly on `<html>` (not just via the Tailwind `dark` class) so there's no gap between this script running and Tailwind's own CSS taking over.

A pure-CSS `#root:empty` branded loader (pulsing "VO"-initials mark + shimmer bar, using `BRAND.initials`) fills the visual gap until React's first paint completes.

`ThemeContext.tsx` itself (full detail in `01-app-shell-routing-auth.md §6`) is the only React Context used for app-wide UI state in this codebase.

## 6. Design tokens & Tailwind config

- **Primary palette**: a violet scale (50–950, `DEFAULT:#7c3aed`) defined in `tailwind.config.js`.
- **Dark-mode surfaces are deliberately NOT in the Tailwind config** — the config file has an explicit comment explaining why: Tailwind can't resolve `rgb(var(--c-xxx))` custom-property references at PostCSS build time without emitting an incorrect hardcoded fallback. Instead, dark surfaces are applied two ways: (a) raw CSS custom properties consumed directly in `index.css` (`--c-card`, `--c-surface`, `--c-va-border`, `--c-muted`, `--wf-bar`), and (b) hardcoded Tailwind utility pairs sprinkled through components (`bg-white dark:bg-[#13141e]` — this exact hex recurs across dozens of components rather than being a named token). **This is a real design-system gap** — a rebuild could fix this properly (e.g. via Tailwind v4's native CSS-variable-based theming, or a small custom plugin) rather than carrying forward the hardcoded-hex-repeated-everywhere pattern.
- `index.css` also defines a `@layer components` set of reusable classes (`.btn`/`.btn-primary`/`.btn-ghost`/`.btn-danger`/`.btn-outline`, `.card`, `.input`/`.label`/`.field-error`, `.badge*` variants, thin/hidden scrollbar utilities, `dropIn`/`shimmer`/`fadeUp` keyframes) — these **are** consistently reused across the app (unlike the JS component-level duplication noted in §3), since they're just CSS classes applied via `className` regardless of which page's markup they're used in.
- Font: Inter (`fontFamily.sans`).

## 7. Brand configuration (`config/brand.ts`)

A single `BRAND` object (name, initials, tagline, domain, legal company name/address, contact emails, social handles), every field overridable via a `VITE_BRAND_*` env var, defaulting to the original product's own branding (`"Voxigo"`, domain `voxigo.app`, etc. — **these defaults should be replaced with the in-house product's actual identity** before any rebuild ships, since they currently reference the original commercial product's name/company/address). Consumed by `Sidebar`, `PageLoader`, legal pages (Terms/Privacy/About), and referenced in a comment in `index.html` (though the actual favicon there is a hardcoded inline SVG with "VO", not brand-driven — a small inconsistency: the brand initials default to "VO" too, so it happens to line up today, but changing `VITE_BRAND_INITIALS` wouldn't actually update the favicon).

## 8. Build & environment config

- **`vite.config.ts`**: dev server on port 5173, proxies `/api` → `VITE_DEV_PROXY_TARGET` (default `http://localhost:4000`); polling-based file watching (300ms interval) specifically for Docker/Windows-volume-mount compatibility — relevant if the rebuild's dev environment also runs inside a container/VM rather than bare-metal. Production build: `es2020` target, `esbuild` minification, no sourcemaps, manual vendor chunk splitting (`vendor-react`, `vendor-query`, `vendor-forms`, `vendor-ui`, `vendor-http`) with deterministic hashed filenames for CDN caching.
- **Env vars** (`VITE_`-prefixed, read via `import.meta.env`): `VITE_API_URL` (prod backend base URL), `VITE_DEV_PROXY_TARGET` (dev-only), the full `VITE_BRAND_*` white-labeling set (§7). A `VITE_PAYPAL_CLIENT_ID` reference exists in billing-service comments — irrelevant once billing is out of scope.
- **TypeScript**: build script is `tsc && vite build` — type-checking gates the production build (a type error fails the build, not just a lint warning).

## 9. Rebuild Notes

### Billing dependency to resolve first
`PlanGate`/`useSubscription` (§2) is the highest-priority billing-coupling item in the entire frontend, since it's referenced from multiple pages. Decide the stub-vs-remove approach (§2) early, since it affects how much rework every `PlanGate`-using page needs.

### Consolidation opportunities (recurring theme across this whole doc set)
Modal markup, skeleton loaders, status-badge color maps, and pagination controls are each implemented **both** as a shared component **and** duplicated ad-hoc per-page. A rebuild's cleanup pass should either enforce the shared versions everywhere or accept the duplication as a deliberate tradeoff — but the current half-and-half state is worth a conscious decision rather than continuing unexamined.

### Design-system gap to fix
Dark-mode surface colors are hardcoded hex values repeated across dozens of components rather than named Tailwind tokens (§6) — a real maintenance cost if the rebuild ever wants to adjust the dark palette.

### Branding defaults
Replace all `BRAND` defaults (`config/brand.ts`) with the in-house product's actual name/domain/company/address before shipping — the current fallback values are the original commercial product's real branding.

### Dependencies
`tailwindcss`, `postcss`, `autoprefixer`, `vite`, `@vitejs/plugin-react`, `typescript`. No changes needed for an in-house rebuild beyond the branding/design-token items above.
