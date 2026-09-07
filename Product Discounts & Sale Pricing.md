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
  omitting/nulling `discountPercent` clears all three.
- The active window is **inclusive of both endpoints** — `discountStartsAt <= now <= discountEndsAt`.
  Both columns are `timestamptz`; "now" is the server instant, so the window means the same
  thing regardless of where the customer is.
- `effectivePrice` equals the rounded discounted price while the discount is active, and
  the base `price` otherwise. It is always present, so a consumer can render
  `effectivePrice` unconditionally without branching.
- A discount **never changes an existing order**. Order lines snapshot the effective price
  at creation time and are immutable afterwards, matching the existing line-item snapshot
  invariant in `Order State Machine`.

## Integration points

- **Cart** — the cart resolves the effective price the same way order creation does. If the
  cart showed the base price and the order charged the discounted one, the total would drop
  at confirmation; the two must agree.
- **Vendor earnings / commission** — unaffected by design. `VendorEarningsService` reads only
  the `order_items` snapshot, never the live product row, so commission is calculated on
  whatever was actually charged. No change needed (see `Vendor Earnings & Commission`).
- **Search index** — no change needed. The discovery grid renders from `GET /products`
  (`ProductDto`), not from the Meilisearch document; the Meilisearch index only backs the
  suggest/autocomplete bar, which shows no prices.

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

### Sharp edge — PATCH clears on omission

Per PD-1's literal acceptance criteria, `PATCH /products/:id` treats the three discount fields as all-or-nothing on the wire, not just at the domain level: a request that omits `discountPercent` **clears** any existing discount (nulls all three), it does not leave it untouched. Both web callers (vendor PD-3, admin PD-4) always send all three fields — `null` when unset — specifically to avoid silently wiping a discount when the form is submitted to edit something unrelated. Any future caller of this endpoint must do the same, or explicitly re-send the current discount fields.

### Verification

- `npm run lint:api` clean.
- `npm run test:api` — 75 suites / 1032 tests.
- `npm run test -w @hb/web` — 85 files / 1277 tests.
- `npm run build` (shared→api→web) green.
- Migration `1788432000000-ProductDiscounts.ts` verified up/down/up on Postgres 16.
- Confirmed by direct source reading that `VendorEarningsService` computes gross from `order_items.unitPrice` alone and never joins `Product` — commission math is unaffected by the discount change.
- code-reviewer verdict: SHIP, zero FAILs.
