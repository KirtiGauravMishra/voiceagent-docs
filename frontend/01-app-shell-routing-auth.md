# 01 — App Shell, Routing & Auth Pages

> Frontend domain doc. Source: `frontend/src/{App.tsx, main.tsx, hooks/useAuth.ts, services/{api,auth.service}.ts, utils/auth.ts, components/layout/{AppLayout,Sidebar,Navbar}.tsx, contexts/ThemeContext.tsx, config/navigation.ts, pages/{LoginPage,RegisterPage,ForgotPasswordPage,ResetPasswordPage}.tsx}`.

## 1. Entry Point (`main.tsx`)

Standard Vite/React 19 bootstrap: `createRoot(...).render(<StrictMode><App/></StrictMode>)`. One notable non-standard piece: a `vite:preloadError` listener that does a **one-time forced hard reload** (guarded by a `sessionStorage` flag so it can't loop) — this handles the case where a browser tab has an old `index.html` open, a new deploy has shipped with new hashed chunk filenames, and the tab tries to lazy-load a chunk that no longer exists on the server (404). Without this, the user would see a permanent broken/blank page after a deploy until they manually refresh; this makes it self-healing.

## 2. Routing (`App.tsx`)

Single router file. Structure, outside-in:
```
<ErrorBoundary>
  <ThemeProvider>
    <QueryClientProvider>
      <BrowserRouter>
        <Toaster />
        <Suspense fallback={<PageLoader/>}>
          <Routes>...</Routes>
        </Suspense>
      </BrowserRouter>
    </QueryClientProvider>
  </ThemeProvider>
</ErrorBoundary>
```

**Every page is lazy-loaded** (`lazy(() => import(...))`), sharing one top-level `<Suspense>` — meaning route transitions between any two pages show the same `PageLoader` fallback while the target chunk downloads, rather than each page managing its own loading boundary.

### Route table (main app only)

| Path | Component | Guard |
|---|---|---|
| `/` | `LandingPage` | redirects to `/dashboard` if already authenticated |
| `/terms`, `/privacy`, `/about`, `/refunds`, `/docs` | static legal/docs pages | none (always public) |
| `/login`, `/register`, `/forgot-password` | auth pages | `Public` (redirects away if already authenticated) |
| `/reset-password` | `ResetPasswordPage` | none — see §5, this page is currently a stub |
| `/widget-demo/:agentId`, `/widget/voice/:agentId` | embeddable widget pages | none (public by necessity — out of scope, not detailed in this doc set) |
| `/dashboard`, `/agents`, `/agents/:id`, `/calls`, `/chats`, `/campaigns`, `/leads`, `/contacts`, `/workflows`, `/workflows/:id`, `/inbound-calls`, `/knowledge-base`, `/knowledge-gaps`, `/voices`, `/reports`, `/billing`, `/settings/*`, `/notifications` | main app pages | `Private` + wrapped in `<AppLayout/>` (nested route with an `<Outlet/>`) |
| `/settings/phone-numbers` | — | redirects to `/inbound-calls` (legacy path kept as a redirect, not a duplicate page) |
| `*` (catch-all) | — | redirects to `/dashboard` if authenticated, else `/` |

### Guards (all plain components, not a router-level abstraction)
- **`Private`** — `!isAuthenticated()` → redirect to `/login`, else render the wrapped route.
- **`Public`** — the inverse, used on login/register/forgot-password: if already authenticated, redirect to `getHomeRoute()`.
- **`getHomeRoute()`** — resolves the post-login landing route (`/dashboard`). Read directly from the locally-stored user object (`getStoredUser()`), not re-fetched from the server.

## 3. React Query configuration

```ts
new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => (4xx ? false : failureCount < 2),
      staleTime: 30_000,
      gcTime: 5 * 60_000,
      refetchOnWindowFocus: false,
      refetchOnReconnect: true,
    },
    mutations: { retry: 0 },
  },
})
```
4xx errors never retry (client error, retrying won't help); other errors retry up to twice. **Adoption across the codebase is partial** — only a couple of hooks (documented in `12-shared-components-state-styling.md`) actually use `useQuery`/`useMutation`; most pages fetch data with plain `useState`/`useEffect`. Worth deciding during a rebuild whether to fully commit to React Query everywhere (more consistent, better cache behavior) or drop it in favor of the simpler pattern most pages already use (less to learn, but no caching/dedup benefits) — right now the codebase pays React Query's bundle cost without getting its full benefit.

## 4. Auth flow

### Token storage (`utils/auth.ts`)
Plain `localStorage`, three keys: `va_access`, `va_refresh`, `va_user` (the user object itself, not just tokens — avoids a round-trip to re-fetch "who am I" on every page load). No cookies — `api.ts` sets `withCredentials:false` explicitly, since the JWT travels in the `Authorization` header, which works identically whether frontend and backend are same-origin or on separate domains (relevant if the rebuild deploys frontend/backend separately on Replit).

### Axios client (`services/api.ts`)
- **Base URL**: dev uses the Vite proxy (`/api` → whatever `VITE_DEV_PROXY_TARGET` points at, see `12-shared-components-state-styling.md`); production requires `VITE_API_URL`, with a normalizer that strips a trailing `/api` if present so both `https://host.com` and `https://host.com/api` work as the configured value.
- **Request interceptor**: attaches `Authorization: Bearer <token>` from storage; strips the `Content-Type` header for `FormData` bodies (lets the browser set the correct multipart boundary automatically — required for any file-upload endpoint); stamps every request with a `X-Request-ID` (`crypto.randomUUID()`) matching the backend's request-tracing convention (`../backend/01-auth.md`/`../backend/11-app-bootstrap-infra.md`).
- **Response interceptor**:
  - Network/timeout errors are rewrapped into a friendlier `Error` with `isNetworkError:true` for calling code to detect.
  - **403 containing "suspended"** in the message → treated as an account-suspension signal: clears auth, stashes the message in `sessionStorage.auth_notice` (read once by `LoginPage` on next load, see §5), hard-redirects (`window.location.href`, not a React Router navigate — deliberate full-page reload to guarantee a clean app state) to `/login`, and returns a **never-resolving promise** so the original caller's `.catch()` never fires (prevents a duplicate/confusing error toast on top of the redirect).
  - **401** → single-flight token refresh: first 401 sets `isRefreshing=true` and POSTs `/auth/refresh`; any 401s that arrive while a refresh is already in-flight are queued (`failedQueue`) and replayed with the new token once it arrives, rather than each firing their own redundant refresh call. A request already retried once (`_retry` flag) is never retried a second time. Refresh failure clears auth and redirects to `/login`.
- **`getApiErrorMessage(error, fallback)`** — the canonical helper for turning any Axios/API error into toast-friendly text; used throughout the app's pages/hooks.

### `useAuth()` hook
Wraps `authService` (thin Axios wrappers, one call per backend auth endpoint — see `../backend/01-auth.md` for the endpoints themselves) and branches on the shape of the response:
- `{requiresEmailVerification:true}` → returns `{step:'verifyEmail', email}` to the caller, which shows an OTP-entry UI in place (not a route change).
- `{requiresTwoFactor:true, twoFactorToken}` → returns `{step:'2fa', twoFactorToken}`, same in-place-UI pattern.
- Otherwise (a full `{user, accessToken, refreshToken}`) → `storeAuthResult()` persists everything, shows a success toast, and **navigates** to the post-login home route (`/dashboard`).

`logout()` posts the refresh token to `/auth/logout` (best-effort — failure is swallowed, since the client-side cleanup should happen regardless of whether the server-side revocation succeeded), then always clears local storage and navigates to `/login`.

### Auth pages
- **`LoginPage`** — a single component with **three in-place view states** (not three routes): credentials form → (conditionally) 2FA code entry → (conditionally) email-verification code entry, switching via local `useState`, not navigation. Reads a one-shot `sessionStorage.auth_notice` banner (set by the axios interceptor's suspended-account handling, or potentially by other flows) on mount and clears it immediately after showing it.
- **`RegisterPage`** — same pattern minus the 2FA branch (a brand-new account can't have 2FA enabled yet); handles the post-register email-verification step in place.
- **`ForgotPasswordPage`** — requests a reset-code email via `apiClient` directly (no dedicated service file for this one call).
- **`ResetPasswordPage`** — **currently a stub**: per the earlier frontend research pass, this page immediately redirects to `/forgot-password` rather than implementing a working reset-by-link/code flow. Confirm during rebuild whether a real password-reset-completion UI is needed (the backend endpoint `POST /api/auth/reset-password` — see `../backend/01-auth.md §5.6` — is fully implemented and ready; only the frontend UI for it is missing/stubbed).

## 5. Layout Shell

### `AppLayout` (`components/layout/AppLayout.tsx`)
The wrapper every `Private` route renders inside (via nested `<Route element={<Private><AppLayout/></Private>}>` + `<Outlet/>`). Fixed `Sidebar` + `Navbar`, with the main content area's `margin-left`/height computed from the sidebar's collapsed state (persisted to `localStorage['va_sidebar']`) so collapsing the sidebar smoothly reflows the layout via a CSS transition rather than a re-render jump.

### `Sidebar` + `config/navigation.ts`
Purely data-driven: `NAV_CONFIG` is an array of sections (`main`, `settings`), each a list of `{label, path, icon}` items using `lucide-react` icons. Adding/removing/reordering a sidebar item is a config-file edit, not a component change. `getNavLabel(pathname)` reverse-looks-up the current route's label for the Navbar's page title, with a few manually-added special cases (`/settings/profile` → "My Profile", etc.) for routes not directly in `NAV_CONFIG` (redirected/aliased paths).

Two sidebar items map to **out-of-scope backend domains** and should be reconsidered if those domains are dropped: `Subscription` (→ `/billing`) and any settings items tied to billing-gated features. The `Docs` and `Alerts`/`Services`/`API Keys` settings items are all main-app-relevant and stay.

### `Navbar`
Renders: page title (from `getNavLabel`), a search button opening a **client-side-only** `SearchModal` (fuzzy-filters a small hardcoded `NAV_ITEMS` array — this is NOT a real search API, just a command-palette-style nav shortcut, filtered entirely in the browser), a docs shortcut, theme toggle, a notifications bell (dropdown showing the latest 6 notifications, polling-driven unread badge via `useUnreadNotifications()` — see `12-shared-components-state-styling.md` and `../backend/08-notifications-settings.md`), and a profile dropdown (View Profile / Account Settings / Sign out).

## 6. Theming (`contexts/ThemeContext.tsx`)

Minimal context: `{theme: 'light'|'dark', toggle()}`, persisted to `localStorage['vx_theme']` (**defaults to `'dark'`** if unset). `applyTheme()` toggles the `dark` Tailwind class on `<html>` and also sets `colorScheme` + inline `background-color`/`color` directly — the inline styles exist specifically to avoid a flash-of-wrong-theme gap between whatever a pre-hydration script in `index.html` set and React actually mounting (see `12-shared-components-state-styling.md` for the full pre-hydration-script + build-config picture).

## 7. Rebuild Notes (Replit target, in-house/no-billing)

### Billing-gated UI to remove
Remove the `/billing` route, sidebar nav item, and any `PlanGate`-gated UI (see `12-shared-components-state-styling.md`) once billing is dropped.

### Fix during rebuild
- **`ResetPasswordPage`** — implement the real reset-by-code completion flow; the backend endpoint already supports it (see §5).
- **React Query adoption** — decide fully-adopt vs. fully-drop rather than the current half-and-half state (§3).

### Dependencies
`react-router-dom` v6, `@tanstack/react-query`, `react-hot-toast`, `axios`, `react-hook-form` + `@hookform/resolvers/yup` + `yup` (auth forms), `lucide-react` (icons). No changes needed for an in-house rebuild — none of these are billing/panel-specific.
