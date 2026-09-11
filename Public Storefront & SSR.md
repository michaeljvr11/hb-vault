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
- Admin/vendor portal render modes � stay `RenderMode.Cli1. **Public `GET /vendors/:id` – approved-only filter confirmed?** Does `VendorsService.findOne`
   already filter to `approved`? If not, add it – or a direct id lookup leaks non-approved vendors.
   ✓ **Resolved (slice 3, 2026-06-26):** approved-only filter added to `findOne` query; non-approved vendors return 404 matching `findDirectory` visibility.

2. **Prerender vs Server:** Recommendation is `RenderMode.Server` for `/shop` and data-driven
   pages (live data + SEO), `Prerender` reserved for future static marketing pages. Confirm.
   ✓ **Resolved (LSM-2/LSM-3, 2026-08-15):** `/about` and `/services` are the static marketing pages; both now carry explicit `RenderMode.Prerender` entries in `app.routes.server.ts`.

3. **Default post-login destination (no returnUrl):** keep `/shop`? Current behaviour is fine.

4. **Nav auth-awareness scope:** minimum = sign-in/account toggle + anonymous cart → login.
   Anything more (mini-cart, account dropdown) is cart-card territory.ogin.
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
- This was the **last slice — the Public StorefrontThe `/products/:id` route component shipped in PR #26 (see [[Category Taxonomy & Discovery]] implementation notes for cards b4VoyjRu + 6qlkwk75). It is fully SSR-rendered, public, and consumes the category/vendor/search discover API. Vendor-profile page (`/vendors/:id`) shipped in PR #35 (card #40 UG5UFyxy); see slice 5 notes below.

### Slice 5 — Public vendor profile page (2026-07-20, UG5UFyxy)

**What shipped:**
- New standalone Angular component `PublicVendorProfile` at `apps/web/src/app/features/vendors/vendor-profile/` (plural `vendors/` — deliberately separate from the vendor-portal `features/vendor/pages/vendor-profile/`). Signals, OnPush, `:id`-param-driven, SSR-safe (HttpClient relative URLs, no browser globals).
- Fetches the vendor via `VendorsService.getById` (public `GET /vendors/:id`, approved-only, shipped in slice 3 / PR #21) and its listings via `ProductsService.list({ vendorId })`. Vendor states loading/loaded/not-found(404)/error; product-grid states loading/loaded/empty/error.
- Hero: business name, optional trading name, a "Verified SME" badge (every vendor the public endpoint returns is APPROVED, so all are verified), country label from `countryCode`. Category-tag chips **derived client-side** (distinct product categories across the vendor's listings, de-duped by id). Product grid reuses the shared `ProductCard`. Anonymous add-to-cart routes to `/login?returnUrl=…` (the established slice-2 boundary).
- Route `/vendors/:id` is public (no guard), placed before the `**` catch-all in `app.routes.ts`, and `RenderMode.Server` in `app.routes.server.ts`.
- Re-pointed the three "visit this vendor" affordances from `/discover?vendorId=` → `/vendors/:id`: product-detail `viewStorefront()`, the discover omnibox "Vendors" suggestion, and the shop-home vendor-showcase tiles (`shop.onVendorSelect`).
- Design artifact authored (no Claude Design source, login unavailable): `docs/design/claude-design/screens/vendor-profile/index.html` + mirror `docs/design/vendor-profile/export.html`.

**Key decisions:**
- **No `@hb/shared` / API change.** The public `VendorResponseDto` returns only `{ id, businessName, tradingName?, status, countryCode }`, so the page is built from those + client-derived category chips. A free-text "about" section was intentionally omitted rather than expand the contract (the card scoped this as web+design only).
- Re-point scope was clarified with the developer: **all three** affordances move to the new page; the discover `?vendorId=` filter machinery is left intact so shared/deep-linked `/discover?vendorId=` URLs still work.
- Component named `PublicVendorProfile` (renamed from `VendorProfile` during review) to avoid a duplicate class name with the vendor-portal page.

**SSR safety:** param-driven from the route, no `window`/`localStorage`/`document`, follows the discover/PDP pattern; `RenderMode.Server`.

**Tests & build:** new `vendor-profile.spec.ts` (happy path incl. products fetched by `vendorId`, not-found via 404 + missing id, generic error, empty state, category-chip de-dupe, country-label mapping, anonymous add-to-cart→/login gate) + updated `product-detail`/`discover`/`shop` re-point specs. `npm run test -w @hb/web` 600/600; `npm run test:api` 406/406 (API untouched); `npm run lint:api` clean; `npm run build` clean (only the pre-existing `admin-catalog.scss` budget warning).

**Code review outcome:** SHIP, no FAILs. Applied two NITs (rename to `PublicVendorProfile`; SME-badge border derived from `--hb-primary` via `color-mix` instead of a hardcoded rgba). Left the `route.paramMap.subscribe` `takeUntilDestroyed` NIT to match the existing `product-detail` sibling pattern (route paramMap completes on teardown — no real leak).

**PR:** #35 (https://github.com/michaeljvr11/hb-mono-repo/pull/35) — open, awaiting human merge.

**Follow-ups:** `docs/design/vendor-profile/reference.png` screenshot not captured (Docker Desktop wasn't running locally, so the full stack couldn't be brought up for a live SSR screenshot); optionally list `vendor-profile` as implemented in `apps/web/CLAUDE.md` + the `DESIGN.md` screens table.

### SSR API Routing — Internal Hairpin & Throttle Identity (2026-09-11, 83G6w4yZ)

**Why this card is in this note:** The storefront spec covers public visibility + SSR enablement, but no infrastructure spec exists yet. This card fixes a critical defect in the SSR → API boundary introduced when the `/shop` route was restored to `RenderMode.Server` in slice 1. Rooted here rather than duplicated into a future infra note.

**The defect:** During SSR the web server called `https://dev.hb-ecommerce.com/api` (the public URL). This request hairpinned through NAT back to Caddy, which proxied it internally to the `api` container. `ThrottlerModule` keys on `req.ip`, and with `trust proxy 1` Express resolves that to the *last* `x-forwarded-for` entry. Every hairpinned SSR request arrived from the same NAT'd address, so that one address was the throttle identity for every server render by every visitor — the entire site's catalogue routes (`/shop`, `/discover`, `/products/:id`, `/vendors/:id`) shared **one 120 req/min throttle bucket** — roughly 120/(calls-per-render) renders/min site-wide, invisible in dev/CI/tests. Fixing the destination alone does not fix identity; verbatim `x-forwarded-for` forwarding is load-bearing.

**What shipped:**
- `apps/web/src/app/core/http/ssr-api-interceptor.ts` (new) — functional `HttpInterceptorFn`. Rewrites SSR calls from `environment.apiBaseUrl` onto `INTERNAL_API_BASE_URL` (an injectable token, null in the browser) and **forwards `x-forwarded-for` verbatim** — no append. One file replaces `isPlatformServer` checks at 91 call sites across 44 files.
- `apps/web/src/app/core/http/internal-api-base-url.token.ts` (new) — `INTERNAL_API_BASE_URL` token + `parseInternalApiBaseUrl` boot-time validation (rejects malformed schemes, missing paths, and aborts before port bind).
- `apps/web/src/app/app.config.ts`, `app.config.server.ts`, `server.ts` — token provisioning (server-only via `app.config.server.ts`; browser gets unconditional pass-through).
- `docker-compose.prod.yml` — `INTERNAL_API_BASE_URL: http://api:3000/api` on `web` service.
- `apps/web/src/app/core/http/ssr-api-interceptor.spec.ts` — 14 specs + mutation test (appending XFF fails exactly one spec).
- `apps/web/CLAUDE.md` — SSR gotchas bullet point added.

**Key decisions:**
- **Verbatim XFF, never appended.** Tested directly: verbatim forward → `req.ip` = real client; appended → `req.ip` = proxy address, collapses throttle bucket. The single most load-bearing detail.
- **Registration in `app.config.ts`, not `app.config.server.ts`.** Angular exposes no public API for functional interceptor contributions from a separate config. Server-only provisioning via the token means the browser gets the unconditional pass-through.
- **Transfer-cache origin split (caught in code review).** `provideClientHydration()` + user interceptors = server caches under internal URL, browser looks up public URL → guaranteed miss → double-fetch billed to real throttle. Fixed with `HTTP_TRANSFER_CACHE_ORIGIN_MAP`, Angular's purpose-built token for a server/client origin split. Because that map is origin-only, the internal URL must keep the same `/api` suffix as the public one.
- **Anchored prefix match** (separate review finding) so a same-origin sibling like `/api-docs` or `/apiary` isn't dragged onto `api:3000` — Caddy routes only `/api/*` there.
- **Malformed config aborts at boot.** `INTERNAL_API_BASE_URL=api:3000/api` (missing scheme) parses as `api:` scheme, yields origin `"null"`, silently breaks the cache again. `=/api` (missing host) throws deep in module init. Both now abort with a message naming the variable, before the port binds.

**Verification (complete locally; post-deploy checks remain open):**
- Express `trust proxy 1` probed directly for all four XFF shapes.
- Mutation test: appending to XFF fails exactly one spec — the verbatim one.
- End-to-end against the built SSR server: `/shop` rendered 98KB with `ng-server-context="ssr"`, all 4 API calls hit `http://api:3000/api` carrying `x-forwarded-for: 203.0.113.7` verbatim, no public URL in output, clean stderr.
- Transfer-cache proof: rendered `/shop` against two different internal base URLs, diffed embedded `ng-state` keys. Before the fix every key differed; after, byte-identical (computed from the public origin the browser uses).
- All four config paths on the real built server: correct URL → SSR + verbatim XFF; unset → public URL, renders fine; both typos → boot aborted, message names the variable.
- Browser bundle checked: no `process.env` reads, no origin-map provider leaked.
- `npm run test -w @hb/web` → 1291 green, 86 files. `npm run build` → clean.

**Tests & build:**
- `npm run test -w @hb/web` → 1291/1291 passed (+14 new in `ssr-api-interceptor.spec.ts`). `npm run build` → clean.
- API suite and `lint:api` deliberately not run locally: this card touches no `apps/api` or `libs/shared` code. CI re-runs both as the PR gate.

**Code review outcome:** First pass returned **NO-SHIP** on one blocking FAIL — the transfer-cache origin split, an undetected double-fetch on every catalogue page that neither the unit tests nor the end-to-end run had caught. That was the most valuable finding of the card. It was fixed and the fix independently verified (see the transfer-cache proof above), along with two WARNs (anchored prefix match, an inaccurate trust-boundary comment). The confirming re-review pass was cut off by a session rate limit and did not complete; its focus points were instead verified directly — origin-map direction against Angular's own token docs, the unset path, browser-bundle leakage, and the `new URL()` crash path (which surfaced the boot-validation gap fixed in commit 4). Worth a fresh review pass before merge.

**PR:** (https://github.com/michaeljvr11/hb-mono-repo/pull/...) — branch `feat/83G6w4yZ-ssr-internal-api-routing`, 6 commits, open.

**Acceptance criteria — post-deploy verification required:**
- ✓ SSR API calls now hit the internal address (`http://api:3000/api` not the public hairpin).
- ✓ `x-forwarded-for` forwarded verbatim (not appended or replaced).
- ✓ Transfer-cache keys computed from the public origin (no double-fetch).
- ✗ Absence of hairpinned requests in Caddy access log during a `RenderMode.Server` render — **must verify on the deployed dev box**.
- ✗ Load test: N concurrent `/shop` renders (N × calls-per-render >> 120/min) yield zero 429s — **must verify on the deployed dev box**.

All other acceptance criteria met locally. The two above are deploy-time checks for the operator.

### HTML-origin security headers come from Caddy, not helmet (2026-09-11, SEC-1 / SEC-2)

Recorded here because it is a property of the SSR serving path, not of auth. Full detail in
[[Auth & Roles]] (SEC-1..SEC-4 note).

- **The SSR HTML origin gets its security headers from the Caddy edge, not from `helmet`.**
  `helmet()` runs inside the NestJS process, which serves only `/api/*` and `/uploads/*`. The
  Angular SSR app is a separate container behind the same hostname, so it never received any of
  helmet's headers. Verified live on 2026-09-11: `/shop` carried no CSP, no `X-Frame-Options`
  and no `Permissions-Policy`, while `/api/health` carried all three. SEC-1 adds
  `Content-Security-Policy "frame-ancestors 'none'"`, `X-Frame-Options DENY`, a
  `Permissions-Policy` and `-X-Powered-By` at the edge.
- **Those headers are scoped to the web `handle` in the `Caddyfile`, and that placement is
  load-bearing.** The site-wide `header` block uses `defer`, which **replaces** rather than
  appends. A site-wide `Content-Security-Policy` would therefore overwrite the full policy
  helmet sets on `/api/*` (`default-src 'self'`, `object-src 'none'`, `script-src 'self'`, …)
  with nothing but `frame-ancestors` — a downgrade of the API's headers. Scoping to the web
  handle leaves helmet untouched and covers only the pages it never reached.
- **The non-production gate now sits in front of the SSR app.** Basic auth plus
  `X-Robots-Tag: noindex, nofollow` at the edge, env-driven, for the dev box only. `/api/health`
  is exempt. Relevant to SSR because there is no `robots.txt` — `/robots.txt` 302s into the
  Angular catch-all, and `RenderMode.Server` serves ~98KB of fully rendered HTML, which makes
  the box ideal crawler food.
- Nonce-based CSP for the HTML origin (SEC-5, card `YYjbfc65`) is **not** shipped — it risks
  breaking Angular hydration and needs its own live-browser verification pass.
