# ADR-004: Android App Links and Deep Link Architecture

**Date:** 2026-04 | **Status:** Accepted

---

## Context

CrewTask for Android needs two distinct "click-to-open-app" mechanisms:

1. **App Shortcuts** - long-press on launcher icon opens My Day, Map, or AI Assistant directly
2. **Email links** - task notification emails (assigned, due soon, overdue) should open the app instead of a browser

Both rely on Android App Links (verified HTTPS deep links). The alternative - custom URL schemes like `com.crewtask.v2://` - works without verification but shows an ugly disambiguation dialog and cannot be used in email clients that strip non-HTTPS links.

Three bugs were fixed in the process of making this reliable (2026-04-14):
- **BUG-NAV-001:** After a cold start via shortcut, tapping the navbar reopened the AI modal instead of navigating
- **BUG-AUTH-001:** Opening a shortcut while logged out showed a broken empty screen instead of a login prompt
- **BUG-LINK-001:** Email "View task" links opened in the browser instead of the app

---

## Options Considered

### Routing on navigation after shortcut launch

**Option A: No guard on getLaunchUrl()**
The initial implementation called `getLaunchUrl()` inside `useEffect([navigate])`. In React Router v6, `navigate` is recreated on each render (it captures location state). This caused the effect to re-run on every route change, re-reading the cached launch URL and force-navigating back to the shortcut destination on every navbar tap.

**Option B: Module-level guard (chosen)**
A module-level boolean `launchUrlHandled` ensures `getLaunchUrl()` is called exactly once per app lifetime. A React ref would reset on component unmount; a module variable persists for the lifetime of the JS bundle. Additionally, query params (`?openAI=true`, `?focus=current`) are removed from the URL after processing to prevent reopening on component remount.

### Auth gate for unauthenticated deep link users

**Option A: Redirect to / and lose the intended URL**
Previous behavior. After login always landed on `/dashboard`, ignoring the shortcut destination.

**Option B: returnTo query param in URL**
Pass intended path as `/?returnTo=%2Fmy-day%3FopenAI%3Dtrue`. Pollutes the URL, visible to users, needs sanitization everywhere it's read.

**Option C: Zustand store pendingDeepLink (chosen)**
`ProtectedRoute` saves the intended path to `useAuthStore.pendingDeepLink` before redirecting to `/`. `LandingPage` auto-opens the auth modal when it detects a pending link. After `SIGNED_IN`, `AppRoutes` navigates to the saved path and clears it. No URL pollution, no additional sanitization needed - the path originates from React Router's own route tree.

**Security note:** Open redirect is not possible. `pendingDeepLink` is set only inside `ProtectedRoute`, which is only rendered for routes registered in the React Router config (internal paths). External URLs cannot reach it.

### Email link domain

**Option A: Use www.crewtask.app in email templates**
Initial state. Android App Links are configured for `crewtask.app` (no www). Vercel redirects `www` to non-www with a 301, but Android does not follow redirects when verifying App Links - verification fails for the www host. Email links opened in browser.

**Option B: Add www.crewtask.app to AndroidManifest and assetlinks.json**
Would require serving a separate `assetlinks.json` at `www.crewtask.app/.well-known/` and maintaining two App Links hosts. Not worth it when the fix is one constant.

**Option C: Remove www from email templates (chosen)**
`APP_URL` in `push-dispatcher/email-templates.ts` changed from `https://www.crewtask.app` to `https://crewtask.app`. One source of truth, no extra infrastructure.

### Allowed deep link paths

Initial `ALLOWED_PATHS` covered only `/my-day` and `/map` (shortcut destinations). Task notification emails link to `/tasks?...`. Expanded to include `/tasks` and `/dashboard`, with `startsWith` prefix matching to handle future path-param variants like `/tasks/uuid`.

### Auth token security in email links

Evaluated whether to embed session tokens directly in email links for instant login ("magic link" style). Decision: do not do this.

- Task notification links contain only UUIDs (task/group IDs) - not sensitive, safe in URLs
- Supabase auth emails already use PKCE (RFC 8252): a short-lived, one-time `code` that is useless without the `code_verifier` stored on-device. This is the correct standard for mobile OAuth flows.
- Bare session tokens in URLs are logged by proxies, cached by browsers, and visible in server access logs. Never put them there.

---

## Decision

**App Links architecture:**
- AndroidManifest: `autoVerify="true"` for `https://crewtask.app` (all paths)
- `public/.well-known/assetlinks.json`: debug SHA256 present; release SHA256 to be added from Play Console after signing key is generated
- All internal links (shortcuts, emails, push notification action URLs) use `https://crewtask.app` - no www, no custom scheme

**Deep link routing:**
- `useDeepLink.ts`: module-level guard prevents re-processing `getLaunchUrl()` on re-renders; ALLOWED_PATHS covers `/my-day`, `/map`, `/tasks`, `/dashboard` with prefix matching
- Query params (`?openAI=true`, `?focus=current`) are stripped from the URL after first processing
- `useDeepLinkAuth.ts`: custom scheme `com.crewtask.v2://auth-callback` retained for OAuth PKCE code exchange (separate concern from shortcut navigation)

**Auth gate:**
- `useAuthStore`: `pendingDeepLink: string | null` + `setPendingDeepLink`
- `ProtectedRoute`: saves intended path before redirecting unauthenticated users to `/`
- `LandingPage`: auto-opens auth modal when `pendingDeepLink` is set
- `AppRoutes`: post-login `useEffect` navigates to `pendingDeepLink` and clears it

---

## Consequences

**Gained:**
- Shortcuts work correctly: no navbar "stuck" behavior after cold start
- Unauthenticated shortcut users see a login prompt and land on the intended screen after login
- Email "View task" links open the app on devices with the app installed
- Single domain (`crewtask.app`) used consistently across all link-generating surfaces

**Pending:**
- Release SHA256 must be added to `assetlinks.json` before Google Play release. Without it, App Links work only on debug builds (signed with the debug keystore whose fingerprint is already present). Path: Play Console → Release → Setup → App signing → SHA-256 certificate fingerprint.

**Trade-offs:**
- Module-level `launchUrlHandled` flag is a side effect outside React's lifecycle. It resets only on full app restart (JS bundle reload), which is the correct behavior for "cold start URL, handle once." Unit tests require `vi.resetModules()` in `beforeEach` to get a fresh module instance per test.
- `pendingDeepLink` lives in Zustand memory only - does not survive app process kill. For a killed app, the shortcut re-triggers `getLaunchUrl()` on the next launch anyway, so memory-only storage is sufficient.
