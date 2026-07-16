# Frontend Prompt 01 — App Shell, Routing & Auth Pages

> Read `../00-master-rebuild-guide.md` and `../backend/01-auth.md` first.

## Goal
The scaffolding every other frontend prompt builds inside: entry point, router, auth pages, the authenticated layout shell (sidebar + navbar), and theming.

## Entry point (`main.tsx`)
Standard Vite/React bootstrap (`createRoot(...).render(<StrictMode><App/></StrictMode>)`). Add a **one-time forced hard reload** listener on the Vite `vite:preloadError` event, guarded by a `sessionStorage` flag so it can only fire once: this handles the case where a browser tab has an old build open, a new deploy shipped with new hashed chunk filenames, and a lazy `import()` 404s trying to fetch a chunk that no longer exists. Without this the user sees a permanently broken page after a deploy until they manually refresh.

## App structure (`App.tsx`)
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
Lazy-load every page (`lazy(() => import(...))`) sharing one top-level `Suspense` — route transitions show one consistent loading fallback.

### Route table
| Path | Guard |
|---|---|
| `/` (landing) | redirect to `/dashboard` if already authenticated |
| `/terms`, `/privacy`, `/about` (or whatever static pages you need) | none |
| `/login`, `/register`, `/forgot-password`, `/reset-password` | `Public` — redirect to `/dashboard` if already authenticated |
| `/dashboard`, `/agents`, `/agents/:id`, `/calls`, `/chats`, `/campaigns`, `/leads`, `/contacts`, `/workflows`, `/workflows/:id`, `/inbound-calls`, `/knowledge-base`, `/knowledge-gaps`, `/voices`, `/reports`, `/settings/*`, `/notifications` | `Private`, nested inside `<AppLayout/>` (`<Outlet/>`) |
| `*` catch-all | redirect to `/dashboard` if authenticated, else `/` |

Build two plain guard components (not a router-level abstraction): `Private` (redirect to `/login` if not authenticated) and `Public` (the inverse). `getHomeRoute()` resolves the post-login landing route by reading the locally-stored user object — don't round-trip to the server just to decide where to redirect.

## React Query configuration
```ts
new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => (isClientError(error) ? false : failureCount < 2),
      staleTime: 30_000,
      gcTime: 5 * 60_000,
      refetchOnWindowFocus: false,
      refetchOnReconnect: true,
    },
    mutations: { retry: 0 },
  },
})
```
Per the master guide's state-management instruction: **pick one pattern and use it everywhere** — either fully commit to React Query for all server-state (recommended, gives you caching/dedup for free) or use plain `useState`/`useEffect` fetching everywhere. Don't mix both halfway across the app.

## Auth flow

### Token storage
Plain `localStorage`, three keys: access token, refresh token, and the user object itself (avoids an extra round-trip to re-fetch "who am I" on every page load). No cookies — the JWT travels in the `Authorization` header (works identically whether frontend/backend are same-origin or split).

### Axios client
- **Base URL**: dev uses a Vite proxy to the backend; production reads `VITE_API_URL`, normalized to strip a trailing `/api` if present.
- **Request interceptor**: attach `Authorization: Bearer <token>`; strip `Content-Type` for `FormData` bodies (let the browser set the multipart boundary); stamp every request with an `X-Request-ID` (matches the backend's request-tracing convention).
- **Response interceptor**:
  - Rewrap network/timeout errors into a friendlier error with an `isNetworkError` flag.
  - **401 handling — single-flight refresh**: on the first 401, set an `isRefreshing` flag and POST the refresh endpoint; any 401s arriving while a refresh is already in-flight get queued and replayed with the new token once it resolves, instead of each firing a redundant refresh call. Never retry a request more than once. On refresh failure, clear auth and redirect to `/login`.
  - A single `getApiErrorMessage(error, fallback)` helper used everywhere to turn any API error into toast-friendly text.

### `useAuth()` hook
Thin wrapper over one Axios call per backend auth endpoint (Prompt `backend/01-auth.md`), branching on response shape:
- `{requiresEmailVerification:true}` → return `{step:'verifyEmail', email}`, show an OTP-entry UI in place (no route change).
- `{requiresTwoFactor:true, twoFactorToken}` → return `{step:'2fa', twoFactorToken}`, same in-place pattern.
- Full `{user, accessToken, refreshToken}` → persist everything, toast success, navigate to `/dashboard`.

`logout()` posts to the logout endpoint best-effort (swallow failure — client-side cleanup happens regardless), then always clears storage and navigates to `/login`.

### Auth pages
- **`LoginPage`** — one component, **three in-place view states** via local `useState` (not three routes): credentials → 2FA code (if required) → email-verification code (if required). Read a one-shot session-storage notice banner on mount (for a suspended-account message set elsewhere) and clear it immediately after showing it.
- **`RegisterPage`** — same pattern minus the 2FA branch (a new account can't have 2FA yet); handle post-register email verification in place.
- **`ForgotPasswordPage`** — requests a reset code/email.
- **`ResetPasswordPage`** — implement the **real** reset-by-code completion flow (form for code + new password, calling the backend's reset-password endpoint) — don't stub this out as a redirect back to forgot-password.

## Layout shell

### `AppLayout`
Wraps every `Private` route (`<Route element={<Private><AppLayout/></Private>}>` + `<Outlet/>`). Fixed `Sidebar` + `Navbar`; main content area's margin/height computed from the sidebar's collapsed state (persisted to `localStorage`) with a CSS transition on collapse/expand rather than a layout-jump re-render.

### `Sidebar` + nav config
Purely data-driven: a `NAV_CONFIG` array of sections (e.g. `main`, `settings`), each a list of `{label, path, icon}`. Adding/removing/reordering a sidebar item should be a config-file edit, not a component change. A `getNavLabel(pathname)` reverse-lookup drives the Navbar's page title, with a few manually-added special cases for aliased/redirected routes not directly in the nav config.

### `Navbar`
Page title (from `getNavLabel`), a client-side-only command-palette search (fuzzy-filters a small hardcoded nav-items array — not a real search API), a theme toggle, a notifications bell (dropdown of the latest few notifications + polling-driven unread badge, Prompt `backend/08-notifications-settings.md`), and a profile dropdown (View Profile / Account Settings / Sign out).

## Theming
Minimal context: `{theme: 'light'|'dark', toggle()}`, persisted to `localStorage`, **defaults to `'dark'`**. Toggle the Tailwind `dark` class on `<html>` and also set `colorScheme` + inline background/text color directly — the inline styles close the flash-of-wrong-theme gap between a pre-hydration script in `index.html` and React actually mounting.

## Environment variables
`VITE_API_URL` (production API base URL).

## Dependencies
`react-router-dom` v6, `@tanstack/react-query`, `react-hot-toast`, `axios`, `react-hook-form` + `@hookform/resolvers/yup` + `yup`, `lucide-react`.

## Acceptance checklist
- [ ] An unauthenticated user hitting any `/dashboard`-style path is redirected to `/login`; an authenticated user hitting `/login` is redirected to `/dashboard`.
- [ ] Login → 2FA → success all happen without a full route change (verify via the URL bar staying on `/login` until final success).
- [ ] Two API calls that both 401 simultaneously trigger exactly one refresh request, not two.
- [ ] A hard-refresh on any deep route (e.g. `/agents/123`) loads correctly, not a 404 (verify your Replit static-serving/rewrite config handles client-side routing).
- [ ] Theme persists across a reload with no visible flash of the wrong theme.
- [ ] `ResetPasswordPage` completes a real password reset end-to-end against the backend endpoint.
