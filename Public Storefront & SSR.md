# Public Storefront & SSR

Front-of-funnel spec. Implementation flows through `/ship-card` � no code here.
Related: [[HB Domain Model]] � [[Auth & Roles]] � [[Listing Types & Vendor Rules]] � [[Vendor & Admin Portals]] � [[Order State Machine]].

## Problem

The storefront must be publicly browsable with no login. Today it is not:

- `/shop` carries `canActivate: [authGuard]` (`apps/web/src/app/app.routes.ts`) � an
  anonymous visitor is bounced to `/login`. This is a regression: the storefront shipped in
  PR #4 reads the public `GET /vendors/directory` + public products/categories, so it was
  always meant to be anonymous-friendly. The guard defeats its own data layer.
- `app.routes.server.ts` pins `shop` to `RenderMode.Client` � a workaround for the guard,
  not an intrinsic property of the page. A client-only storefront forfeits SSR SEO and
  first-paint benefits that a public catalogue needs.
- The `authGuard` redirect drops the page the user was on � no `returnUrl` is attached.
  Both `Login` and `Register` already read and honour `returnUrl` on success, but nothing
  ever sets it.

Goal: storefront (shop, product listings, product detail, vendor profiles) is fully public
and server-rendered; authentication is required only at the add-to-cart / checkout / order
boundary; when a gate fires the user returns to the page they came from.

## Scope

1. Remove `authGuard` from `/shop` so anonymous visitors can browse.
2. Restore storefront routes to `RenderMode.Server` (or `Prerender` for genuinely static
   pages). Admin (`/admin/**`) and vendor (`/vendor/**`) portals stay `RenderMode.Client`
   (their guards depend on `localStorage`, absent on the server). Do not touch them.
3. **returnUrl capture:** `authGuard` (and `roleGuard`) attach the attempted URL as a
   `returnUrl` query param when redirecting to `/login`, so the existing Login/Register
   return logic receives a value.
4. **Public API parity for SSR:** every server-rendered endpoint must be reachable without a
   token. Products, categories, and the vendor directory are already `@Public()`.
   `GET /vendors/:id` is NOT � the global `JwtAuthGuard` makes it require a token, so an
   SSR-rendered vendor-profile page would 401. Add `@Public()` (approved-only).
5. **Auth-aware storefront nav:** the nav bar is fully static today � no sign-in link for
   anonymous users, no account/sign-out for logged-in ones. Make it reflect auth state and
   route anonymous add-to-cart / cart clicks to `/login?returnUrl=�`.

## Business rules it must honour

From [[Auth & Roles]] and [[Listing Types & Vendor Rules]] � enforce, do not reinvent:

- Auth flow is settled � JWT access + rotating httpOnly refresh cookie, global guards,
  `@Public`/`@Roles`. Extend `AuthService`/guards; never fork the flow.
- Browse is public; transact is authenticated. The auth boundary sits at add-to-cart ?
  checkout ? place order.
- Only `approved` vendors are public-facing. A public `GET /vendors/:id` must not leak
  pending/rejected/suspended vendors. Add an approved-only filter; return 404 for others.
- No PII leak via the public DTO. The `:id` endpoint must stay on `VendorResponseDto`,
  never `AdminVendorResponseDto`.
- SSR safety is non-negotiable. Browser-only APIs stay `isPlatformBrowser`-guarded. New
  auth-aware nav logic follows the `AuthService` pattern.

## @hb/shared contract impact

None expected. This reuses existing contracts (`ProductDto`, `CategoryDto`, `VendorDto`).
No new endpoint is created (`GET /vendors/:id` already exists; only its visibility changes),
so no new DTO is required.

## Out of scope

- Cart & checkout logic � `cart`/`orders` are domain skeletons; this spec only wires the
  auth boundary (anonymous cart action ? `/login?returnUrl=�`), not the cart itself.
- `isVerified` order gate � already its own blocked card; see [[Auth & Roles]].
- New product-detail / vendor-profile route components � designs exist under `docs/design/`;
  this spec makes the route + API public and server-rendered so those screens drop in later.
- Admin/vendor portal render modes � stay `RenderMode.Client`. Do not regress.

## Open questions (confirm before /ship-card)

1. **Public `GET /vendors/:id` � approved-only filter confirmed?** Does `VendorsService.findOne`
   already filter to `approved`? If not, add it � or a direct id lookup leaks non-approved vendors.
2. **Prerender vs Server:** Recommendation is `RenderMode.Server` for `/shop` and data-driven
   pages (live data + SEO), `Prerender` reserved for future static marketing pages. Confirm.
3. **Default post-login destination (no returnUrl):** keep `/shop`? Current behaviour is fine.
4. **Nav auth-awareness scope:** minimum = sign-in/account toggle + anonymous cart ? login.
   Anything more (mini-cart, account dropdown) is cart-card territory.

## Vertical slices ? Trello cards

| # | Title | Card ID |
|---|---|---|
| 1 | Make `/shop` public + restore SSR | kYbVZyqV |
| 2 | Capture returnUrl in guards on redirect | eOI6gKpN |
| 3 | Public `GET /vendors/:id` (approved-only) | vs3y5tDJ |
| 4 | Auth-aware storefront nav | EFXHykuT |


## Implementation Notes

### Slice 1 — Make `/shop` public + restore SSR (2026-06-24, kYbVZyqV)

**What shipped:**
- `apps/web/src/app/app.routes.ts` — removed `canActivate: [authGuard]` from `/shop` route. Anonymous browsing now works.
- `apps/web/src/app/app.routes.server.ts` — flipped `shop` from `RenderMode.Client` to `RenderMode.Server`; corrected stale comment; `/admin/**`, `/vendor/**` remain `RenderMode.Client` (no regression).

**SSR safety verified:**
- Full render chain (Shop → NavBar/Footer → AuthService.initialize()) is `isPlatformBrowser`-guarded.
- Runtime test: `GET /shop` → HTTP 200, `ng-server-context="ssr"`, full storefront HTML rendered server-side, no `/login` redirect, no browser-API crash.

**Tests & build:**
- `npm run test -w @hb/web` → 272/272 passed (incl. shop.spec.ts + auth-guard.spec.ts).
- `npm run build` → clean SSR build.

**Code review outcome:**
- SHIP. No FAILs. Flagged pre-existing follow-up: `shop.ts` `formatPrice` hardcodes `en-ZA` locale + NAD/else-R branch — worth a future storefront card (not introduced here, not a blocker).

**PR:** #19 (open, awaiting human merge).

**Remaining slices:**
- Slice 2 (eOI6gKpN): Capture returnUrl in guards on redirect.
- Slice 3 (vs3y5tDJ): Public `GET /vendors/:id` (approved-only).
- Slice 4 (EFXHykuT): Auth-aware storefront nav.

**Follow-ups:**
- Currency/locale formatPrice card under storefront epic.

### Slice 2 — Capture `returnUrl` in guards on redirect (2026-06-24, eOI6gKpN)

**What shipped (frontend only):**
- `apps/web/src/app/core/auth/auth-guard.ts` — redirect now `createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })`. Uses the `CanActivateFn` `state.url` (the full attempted in-app path, query string included).
- `apps/web/src/app/core/auth/guards/role-guard.ts` — same `returnUrl` capture on its unauthenticated/forbidden redirect to `/login`.
- `apps/web/src/app/core/auth/return-url.ts` (new) — `sanitizeReturnUrl()` open-redirect guard: honours only same-app absolute paths (single leading `/`), rejects `//host`, `/\host`, any-backslash, and non-leading-slash/absolute URLs (`https:`, `javascript:`).
- `apps/web/src/app/auth/login/login.ts` + `register.ts` — the `returnUrl` query param (attacker-controllable from the URL bar) now flows through `sanitizeReturnUrl`; unsafe values fall back to the default destination (`/shop`, or `/admin/dashboard` for admins on login).

**Key decisions:**
- **Capture at the guard, sanitise at the consumer.** `state.url` is inherently same-app (always `/…`), so the producing side needs no sanitising; the open-redirect risk lives only where the param is read back off the URL and fed to `navigateByUrl` — so the single sanitiser chokepoint sits in `return-url.ts`, shared by login + register.
- **`navigateByUrl` keeps navigation inside the Angular router** (never a cross-origin full-page redirect), so the threat model is protocol-relative/backslash/absolute strings — all rejected. Code-reviewer confirmed the exploitable vectors are closed.
- **Extended the settled guards, did not fork the auth flow** (per [[Auth & Roles]]). No `@hb/shared`, API, or migration change.

**Tests & build:**
- `auth-guard.spec.ts` + `role-guard.spec.ts` assert the redirect `UrlTree` carries the expected `returnUrl` (incl. query-string preservation, and both anonymous + forbidden-role paths for roleGuard).
- `return-url.spec.ts` (new) covers the sanitiser (safe paths through; absolute/protocol-relative/backslash/relative rejected).
- `login.spec.ts` + `register.spec.ts` cover the consuming side end-to-end: safe `returnUrl` honoured, external `returnUrl` falls back to default.
- `npm run test -w @hb/web` → 285/285 passed. `npm run build` → clean. `npm run lint:api` → clean.

**Code review outcome:**
- FIX-FIRST → SHIP. One blocking finding: `register.ts` carried the same sanitised navigation as login but shipped without a test; added two register-side cases mirroring login. Remaining NITs (sanitiser passes harmless oddities like `/@host`, `/%2F%2Fhost`) confirmed non-exploitable via the router — left as-is.

**PR:** _(to be linked)_ — open, awaiting human merge.

**Remaining slices:**
- Slice 3 (vs3y5tDJ): Public `GET /vendors/:id` (approved-only filter).
- Slice 4 (EFXHykuT): Auth-aware storefront nav (sign-in link, account toggle, anonymous cart → `/login?returnUrl=…`). Now fully unblocked by this slice's `returnUrl` round-trip.
### Slice 3 — Public `GET /vendors/:id` (approved-only) (2026-06-26, vs3y5tDJ)

**What shipped:**
- `apps/api/src/vendors/vendors.controller.ts` — added `@Public()` to `@Get(':id')` so SSR-rendered vendor-profile pages can fetch without a token. Route ordering already safe — `directory`, `me`, `me/dashboard` declared before `:id`.
- `apps/api/src/vendors/vendors.service.ts` — `findOne` now queries `{ where: { id, status: VendorStatus.APPROVED } }`. Non-approved vendors hit `NotFoundException` → 404, matching `findDirectory` visibility. Response stays on the lean `VendorResponseDto` (no PII leak: `registrationNumber`/`verificationDocumentUrl` not exposed).
- `apps/api/src/vendors/vendors.service.spec.ts` — new `describe('findOne (public vendor profile)')`: approved vendor returned & mapped to public DTO; PII fields omitted; each non-approved status → 404 (parametrised); unknown id → 404. Mock honours the `where` clause so the approved-only filter is genuinely exercised.

**Key decisions:**
- **Approved-only filter in the service (query layer).** Keeps data-access logic centralised; reusable if a future admin route needs a separate scoped lookup.
- **No `@hb/shared` change, no migration.** All fields already exist; this is a visibility + filtering change.

**Tests & build:**
- `npm run test:api` → 112 passed (+6 new in vendors.service.spec.ts).
- `npm run test -w @hb/web` → 285 passed (no web files touched).
- `npm run lint:api` → clean.
- `npm run build` → clean (shared → api → web).

**Code review outcome:**
- SHIP. No FAILs. One non-blocking note: if an admin single-vendor-detail view is ever needed it must be a separate admin-scoped route (this one returns the lean DTO and hides non-approved) — out of scope.

**PR:** #21 (open, awaiting human merge).

**Remaining slices:**
- Slice 4 (EFXHykuT): Auth-aware storefront nav (sign-in link, account toggle, anonymous cart → `/login?returnUrl=…`).

### Slice 4 — Auth-aware storefront nav (2026-06-29, EFXHykuT)

**What shipped (frontend only — no `@hb/shared`, no API, no migration):**
- `apps/web/src/app/layout/nav-bar/nav-bar.ts` + `.html` + `.scss` — the nav bar now reflects auth state via `AuthService.currentUser$`. Anonymous visitors see a **"Sign in"** link → `/login`; authenticated users see their **account name** (first name, falling back to email) + a **"Sign out"** control calling `AuthService.logout()`. Switches reactively on login/logout.
- Anonymous **cart** click routes to `/login?returnUrl=<current url>` (via `Router.url`) — consumes the producing side of the slice‑2 returnUrl round‑trip so users return to the page they were browsing. Authenticated cart keeps the existing "coming soon" snackbar. Search left as "coming soon" (scope kept tight to cart per the card).
- "Sell on H&B" → `/vendor/apply` entry preserved.
- `apps/web/src/app/layout/nav-bar/nav-bar.spec.ts` — 10 Vitest specs (see below).

**SSR / hydration safety (the crux):**
- `/shop` is `RenderMode.Server` and the nav renders server-side in the anonymous state (no token on the server). Because `APP_INITIALIZER` populates `currentUser$` before the client bootstraps, a logged-in user's first client render would otherwise diverge from the anonymous server DOM and throw a hydration mismatch.
- Fix: `toSignal(currentUser$, {initialValue:null})` + a `hydrated` signal flipped only inside `afterNextRender`, with `currentUser = computed(() => hydrated() ? user() : null)`. The server render and the initial client hydration pass are therefore both anonymous; the real auth state swaps in only after hydration.
- No `window`/`localStorage`/`document` touched in the nav.

**Security:**
- `returnUrl` is set to `Router.url` (in-app, not attacker-controlled); `/login` independently sanitizes it via `sanitizeReturnUrl` (open-redirect guard from slice 2). No new vector.

**Tests & build:**
- `npm run test -w @hb/web` → **293 passed (26 files)**; `npm run test:api` → 112 passed; `npm run lint:api` clean; `npm run build` clean (only the pre-existing `shop.scss` / `admin-catalog.scss` budget warnings, unrelated).
- Nav-bar specs cover: anonymous vs authenticated render, email fallback, reactive switch, `signOut()`→`logout()`, anonymous cart→`/login` with returnUrl, authenticated cart does not navigate, and "Sell on H&B" preserved.

**Code review outcome:**
- SHIP. No FAILs. Non-blocking nits noted: (1) the gate-closed/pre-hydration state isn't reliably unit-testable in jsdom (afterNextRender always runs in the browser test env; it simply never runs on the server) — verified by reasoning instead; a pre-hydration test was tried and reverted as misleading; (2) minor mobile a11y — the account name is `display:none` < 768px so only the (aria-hidden) person icon shows.

**PR:** #22 (https://github.com/michaeljvr11/hb-mono-repo/pull/22) — open, awaiting human merge.

**Epic status:**
- This was the **last slice — the Public Storefront & SSR epic is now complete** (slices 1–4 all shipped: public `/shop`+SSR, returnUrl capture, public `GET /vendors/:id`, auth-aware nav).

---

## Cross-note: product-detail & vendor-profile routes

The `/products/:id` route component shipped in PR #26 (see [[Category Taxonomy & Discovery]] implementation notes for cards b4VoyjRu + 6qlkwk75). It is fully SSR-rendered, public, and consumes the category/vendor/search discover API. Vendor-profile page (`/vendors/:id`) design remains pending; follow-up card #40 UG5UFyxy (vendor profile page with stats/reviews) created on the board.
