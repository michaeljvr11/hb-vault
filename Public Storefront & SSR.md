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
