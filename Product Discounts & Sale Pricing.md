# Product Discounts & Sale Pricing

## Problem

There is no way to put a product on sale. A vendor who wants to run a promotion has to
edit the base `price` down and remember to put it back afterwards — which loses the
original price, gives the customer no signal that the product is discounted, and leaves
no record of what the product normally costs. `HB Domain Model` explicitly deferred
vendor-run promotions past Phase 1 launch ("planned as a fast follow"); this is that
fast follow.

## Scope

- A product carries an optional **discount percent** plus an explicit **start/end window**.
  All three fields are set together and cleared together — a discount is never half-configured.
- The discount is **scheduled**, not instantaneous: setting one in the future leaves the
  product at its base price until the window opens, and it lapses on its own when the
  window closes. Nobody has to remember to turn it off.
- The **effective price is resolved server-side** on every read path. The client is never
  handed a raw percent and asked to do the arithmetic — see Decision below.
- **Vendors** set discounts on their own products; **admins** set them on platform
  (first-party) listings. Confirmed with the product owner: v1 covers both, not vendor-only.
- The storefront shows the **crossed-out original price** next to the discounted price and
  an **"X% Off" badge** on the product image, on both the discovery grid and the PDP.
- The discounted price is what the customer is **charged**: it flows through the cart and is
  **snapshotted onto the order line** at order-creation time.

## Decision — server-computed effective price (not a client-side calculation)

The API returns the raw fields (`discountPercent`, `discountStartsAt`, `discountEndsAt`)
**and** two derived fields (`isDiscountActive`, `effectivePrice`) on every `ProductDto`.

The alternative — ship the raw percent and let each surface compute the sale price — was
rejected. Money arithmetic done independently in the product card, the PDP, the cart and
the order summary is four chances to round differently, and "is the discount live right
now?" evaluated against the *browser's* clock is a correctness bug waiting for a customer
in a different timezone with a skewed system clock. One server-side resolution, one
rounding rule, one notion of "now". This is the same reasoning that keeps commission
math server-side in `Vendor Earnings & Commission`.

Rounding is **half-up to 2 decimal places**, consistent with `Money & Currency Rules`.

## Rules

- `discountPercent` is `numeric(5,2)`, strictly greater than `0` and at most **90**
  (confirmed with the product owner; the cap is a fat-finger guard, not a business ceiling).
- `discountEndsAt` must be **strictly after** `discountStartsAt`.
- The three discount fields are **all-or-nothing**: setting a discount requires all three;
  an explicit `discountPercent: null` clears all three. On **update**, omitting all three
  leaves an existing discount untouched — the same "absent = untouched" convention as
  `categoryIds`/`sizes`. A window field sent without a percent is an incomplete triple and
  is rejected, not treated as a clear.
- The active window is **inclusive of both endpoints** — `discountStartsAt <= now <= discountEndsAt`.
  Both columns are `timestamptz`; "now" is the server instant, so the window means the same
  thing regardless of where the customer is.
- `effectivePrice` equals the rounded discounted price while the discount is active, and
  the base `price` otherwise. It is always present, so a consumer can render
  `effectivePrice` unconditionally without branching.
- A discount **never changes an existing order**. Order lines snapshot the effective price
  at creation time and are immutable afterwards, matching the existing line-item snapshot
  invariant in `Order State Machine`.


- **Public/non-owner reads show only the derived fields.** Confirmed with the product owner (2026-09-21): anonymous and non-owner reads (`GET /products`, `GET /products/:id`) return `isDiscountActive`/`effectivePrice` only — the raw `discountPercent`/`discountStartsAt`/`discountEndsAt` triple is limited to owner reads (vendor for their own `vendor` listings, admin for `platform` listings). A scheduled-but-not-yet-live discount is never visible to an anonymous shopper or a competitor. Shipped in the storefront follow-ups batch (card `USLrzcr6`).

## Integration points
- **Cart** — the cart resolves the effective price the same way order creation does. If the
  cart showed the base price and the order charged the discounted one, the total would drop
  at confirmation; the two must agree.
- **Vendor earnings / commission** — unaffected by design. `VendorEarningsService` reads only
  the `order_items` snapshot, never the live product row, so commission is calculated on
  whatever was actually charged. No change needed (see `Vendor Earnings & Commission`).
- **Search index** — now discount-aware. The Meilisearch index carries `effectivePrice` and
  `isDiscountActive`, resolving them the same way every other read path does via
  `resolveEffectivePrice()`. Search results sort/filter/facet on `effectivePrice`, not the
  base `price`. Like `GET /products`, search results show only the derived fields
  (`isDiscountActive`/`effectivePrice`) to non-owners — the raw discount window is never
  visible to anonymous searches. (See card HEbrA8WA.)

## Open questions

None outstanding. The two originally flagged here — the percent cap and whether admin
platform listings were in v1 — were both confirmed with the product owner during the
PD-1..PD-5 build (cap 90; admin listings included).


## Implementation Notes (2026-09-07)

**Branch:** `feat/UdeL4byD-product-discounts`
**Cards:** PD-1 UdeL4byD, PD-2 WY3NxxLs, PD-3 AchyX7xS, PD-4 1ceMr9wa, PD-5 J7NivPZU (5 cards, 1 bundled batch)
**PR:** not yet opened at time of writing.

### What shipped

- **PD-1 (contract + resolver):** `@hb/shared` `ProductDto`/related contracts gain raw `discountPercent`/`discountStartsAt`/`discountEndsAt` plus server-derived `isDiscountActive` and `effectivePrice` (both always present, no branching needed by consumers). Migration `1788432000000-ProductDiscounts.ts` adds the three nullable columns (`numeric(5,2)`, two `timestamptz`) — verified up/down/up against a real Postgres 16. `resolveEffectivePrice()` lives in `apps/api/src/common/utils/product-discount.utils.ts` as the single resolver used by every read path. Rounding is half-up to 2dp done in integer cents/basis-points, not `Math.round(pct * price) / 100` float math. `DISCOUNT_PERCENT_MAX = 90` enforced as a constant.
- **PD-2 (order snapshot):** `OrdersService.create` now snapshots `resolveEffectivePrice()`'s result into `order_items.unitPrice` instead of the raw `products.price`. Reuses the existing `orderCreatedAt` instant so discount-window resolution and the commission-rate snapshot share one clock — a multi-line order can't straddle a discount-window boundary mid-creation.
- **Cart fix (owner-approved, beyond the five cards):** `CartService.toDto` was reading the live base price, so the cart would show full price while the order charged the discounted one. It now resolves through the same `resolveEffectivePrice()` util as orders.
- **PD-3 / PD-4 (vendor + admin UI):** percent + start/end inputs on the vendor product screen and the admin catalog's platform-listing screen, each with a none/scheduled/active/expired state chip driven by `isDiscountActive` + the window fields.
- **PD-5 (storefront):** product card and PDP render a struck original price and an "X% Off" badge using the `--hb-sale`/`--hb-on-sale` design roles. This fills a slot `docs/design/DESIGN.md` had already reserved but marked as rendering nothing "because the API has neither field" — that caveat is now stale and should be dropped next time DESIGN.md is touched.

### Decisions confirmed with the product owner during the build

1. Max discount percent = 90 is a fat-finger guard, not a business ceiling (matches the Rules section above, now shipped as `DISCOUNT_PERCENT_MAX`).
2. Admin-owned platform listings do get discounts in v1 (PD-4 shipped, not closed).
3. The cart resolves the effective price too (see fix above) — this was found during the build, not spec'd up front.

### Corrected 2026-09-10 — PATCH no longer clears on omission

The rule above originally read "`null`/omitted clears", and PD-1 implemented it literally: `PATCH /products/:id` treated the discount fields as all-or-nothing **on the wire**, so a request that omitted `discountPercent` nulled all three. That made renaming a product silently end a live sale, and it contradicted the `categoryIds`/`sizes` "absent = untouched" convention in the same DTO. It was caught in review of the PD-1..PD-5 PR and fixed before merge.

`undefined` and `null` are now distinct signals: omitting all three discount fields leaves an existing discount untouched, and clearing requires an explicit `discountPercent: null`. Sending a window field without a percent is an incomplete triple and returns 400 rather than silently dropping the discount.

Both web callers (vendor PD-3, admin PD-4) still send all three fields — `null` when unset — but now because the form genuinely owns all three, not as a defensive workaround. A new caller of this endpoint no longer has to know anything special.

### Verification

- `npm run lint:api` clean.
- `npm run test:api` — 75 suites / 1032 tests.
- `npm run test -w @hb/web` — 85 files / 1277 tests.
- `npm run build` (shared→api→web) green.
- Migration `1788432000000-ProductDiscounts.ts` verified up/down/up on Postgres 16.
- Confirmed by direct source reading that `VendorEarningsService` computes gross from `order_items.unitPrice` alone and never joins `Product` — commission math is unaffected by the discount change.
- code-reviewer verdict: SHIP, zero FAILs.


## Implementation Notes (2026-09-22) — Storefront follow-ups batch

**Branch:** `feat/USLrzcr6-storefront-follow-ups`
**Cards:** USLrzcr6, HEbrA8WA, aqBWlueZ, PlCJpL7Z (4 cards, 1 bundled batch)
**PR:** not yet opened at time of writing.

### What shipped

**USLrzcr6 (Decision: discount visibility):** `GET /products` and `GET /products/:id` now distinguish owner from non-owner reads. A new `OptionalAuth()` decorator/guard mechanism (`apps/api/src/common/decorators/optional-auth.decorator.ts`, modifications to `apps/api/src/common/guards/jwt-auth.guard.ts`) attempts the JWT strategy without rejecting unauthenticated requests — allowing the endpoints to know who is asking. `ProductToResponseDto` (in `apps/api/src/common/utils/mappers.utils.ts`) gained an optional `viewer` param and an `isDiscountOwner()` helper: owner reads (vendor on their own `vendor` listings, admin on `platform` listings) get the raw `discountPercent`/`discountStartsAt`/`discountEndsAt` triple; non-owner reads get only the derived `isDiscountActive`/`effectivePrice` fields. Anonymous shoppers and competitors never see a scheduled-but-not-yet-live discount.

**HEbrA8WA (Search discount-awareness):** The Meilisearch product search index now carries `effectivePrice` and `isDiscountActive`, resolved via the same `resolveEffectivePrice()` utility used everywhere else. Sort/filter/facet switched from `price` to `effectivePrice`, making live discounts visible in search results. Search results (like `GET /products`) respect the visibility rule above — only derived fields are shown to non-owners. Window-boundary decision: a discount's start/end is a time-based state change with no database write to trigger the existing event-driven index upserts; a 5-minute cron job (`reindexDiscountBoundaries()` in `search-indexer.service.ts`, new) re-indexes only products whose windows fell in the elapsed interval, balancing accuracy (not waiting 3am for a mid-day discount) against cost.

**aqBWlueZ (Ratings on product listings):** `ProductDto` gained optional `averageRating` and `reviewCount` fields, populated via one grouped SQL aggregate per `findAll` page (no N+1). The product card renders a compact "★ 4.6 (128)" next to the category label when `reviewCount > 0`, reusing the existing `roundAverageRating` utility. This was a design-review follow-up (not a vault spec note), traced to `docs/design/redesign/PLAN.md` §5 card 1.

**PlCJpL7Z (One search input on desktop):** The header search bar now fetches real suggestions (mapping logic extracted to `apps/web/src/app/shared/suggestion-mapper.ts`, shared with `/discover`), seeds itself from `/discover`'s current `?q=` if present, and merges (not replaces) existing filters on submit. `/discover` removes its own search bar + category chip row at desktop widths (≥768px) via a new SSR-safe viewport signal in `apps/web/src/app/shared/viewport.ts`, leaving exactly one search input in the DOM. This was also a design-review follow-up (not a vault spec note), traced to `docs/design/redesign/PLAN.md` §5 card 7.

### Decisions confirmed during the build

1. **Window-boundary reindex window for discounts (HEbrA8WA):** Product owner chose 5-minute targeted reindex over storing the raw discount window for query-time computation. The latter would require app-side re-sorting after Meilisearch returned results (Meilisearch cannot sort by derived/computed values), which breaks at scale. 5-minute intervals keep price sort/filter/facet correct for a discount starting mid-day without a separate full-reindex.
2. **Owner identity on previously-fully-public routes (USLrzcr6):** `GET /products`/`GET /products/:id` were `@Public()` with no identity resolution at all. Chose a narrowly-scoped `OptionalAuth()` mechanism (only these two routes attempt-but-never-reject the JWT strategy) over adding a separate authenticated endpoint for vendor/admin edit forms, since the existing vendor/admin product screens already read from these same public endpoints and a new endpoint would have meant a frontend data-source change too.

### Corrected / Clarified

The "Search index" line in the Integration points section (above) previously stated "no change needed ... the Meilisearch index only backs the suggest/autocomplete bar, which shows no prices." This was stale — the index does back product search with price sort/filter/facet (used by the `/discover` route and header search). The line was corrected during this implementation to reflect that search is now discount-aware.

### Verification

- `npm run test:api` — 80 suites / 1105 tests, all pass.
- `npm run test -w @hb/web` — 93 files / 1374 tests, all pass.
- `npm run lint:api` — clean.
- `npm run build -w @hb/shared` and `npm run build -w @hb/api` — both green. The full monorepo `npm run build` (which includes `apps/web`'s build-time prerender) was not run locally — prerendering needs `PRERENDER_API_BASE_URL`/`INTERNAL_API_BASE_URL` pointed at a reachable API (see `apps/web/CLAUDE.md`), which this local shell doesn't have set; this is a pre-existing local-environment limitation, not something this batch changed. CI's build step covers it.
- **Code review:** one blocking FIX-FIRST finding during review of card USLrzcr6 (missing `JwtAuthGuard` direct test coverage for the new `OptionalAuth` path), fixed same session; confirming pass returned SHIP. Two addressable findings on the search/header slices fixed inline; two pre-existing follow-ups noted (a `--hb-secondary` token use on the PDP reviews section, and a pre-existing stale-suggestion race in the omnibox search duplicated to the header) spun off as separate cards, neither blocking.
- Evidence log: 385 commits · 360 AI-tagged (93%) · 173 specs · 30 prod blocks.
