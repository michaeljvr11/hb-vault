# Configurable Shipping Fee

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[Money & Currency Rules]] · [[Order State Machine]] · [[HB Domain Model]] ·
[[Cross-Border & Customs]] · [[Vendor Earnings & Commission]] · [[Vendor & Admin Portals]] ·
[[Analytics & Reporting]].

## Problem
`shippingTotal` has been honestly hardcoded to `0.00` since the checkout one-shot
(PR #27) — see [[Order State Machine]] implementation note: "the `shippingTotal`
column and UI row exist, honestly zero until priced", with "price `shippingTotal`"
listed there as an explicit follow-up. Customers currently pay nothing toward the
ZA→NA delivery leg.

This note specs an **admin-configurable flat shipping fee**, applied to every order
and shown as its own line item at checkout, added to the customer's total.
It is configured the same way the commission rate is
([[Vendor Earnings & Commission]] VE-1): an append-only, effective-dated history,
never a mutable single value.

**Upstream conflict resolved (2026-08-24 — Q1 resolved).** [[HB Domain Model]] records
the delivery-fee *model* as settled at "pure pass-through at cost, no margin"
(`H&B Brain/08-revenue-model.md`), and [[Landing Site Migration]] carries
"shipping + customs passed through" into public marketing copy. A single global flat
fee is not pass-through-at-cost. The business model has been clarified: the flat
fee is an **operational approximation of average pass-through cost**, and the feature
ships as designed. The marketing copy ("shipping + customs passed through") **was
deliberately NOT edited** — a human must review it separately and decide whether to
update it or to retract the claim.

## Scope
In scope: one global flat shipping fee **per route and currency**, admin-configurable via an
effective-dated history; applied at order creation; surfaced as a distinct line item
at checkout and on the order; frozen onto the order as charged. A **route** is an
origin→destination `CountryCode` pair; `CountryCode` has exactly two members (ZA, NA),
so there are exactly **4 routes: ZA→ZA, ZA→NA, NA→NA, NA→ZA**, all configurable.
NA→ZA is included deliberately: `orders.originCountry` is derived from
`product.originCountry` and can legitimately be NA, so a NA→ZA order is representable
and must never resolve to a guessed fee. A fee "set" is therefore **8 rows (4 routes ×
2 currencies), all required, one transaction** (Q4 resolved). **Also in scope (added 2026-08-23):**
an optional per-product shipping-fee override, admin-settable, that beats the global
default for that product. See "Per-product override" below.

## Business rules it must honour

From this vault (existing conventions this feature must not violate):

- **Money is `numeric(12,2)` + an explicit currency column, always. Never floats,
  never an implied currency** ([[Money & Currency Rules]]). The fee is money, unlike
  the commission rate (a currency-free percentage) — this is the one place the
  "mirror commission" instruction cannot be followed literally. See Data model.
- **ZAR and NAD are never summed and the 1:1 peg is data, never an assumption**
  ([[Money & Currency Rules]]). A fee configured in ZAR must NOT be applied to a
  NAD order by assuming parity. Every currency the platform accepts carries its own
  explicitly configured amount.
- **One order = one currency** ([[Order State Machine]]). The fee resolves in the
  order's single currency; no conversion, ever.
- **Order totals are stored as recorded at order time, never recomputed from live
  values** ([[Money & Currency Rules]]). A later fee change must never restate a past
  order's total — the same reasoning that made `commission_rates` append-only in
  [[Vendor Earnings & Commission]].
- **Totals are always computed server-side**; a client-submitted total is never
  accepted (`CreateOrderRequest` doc-comment, `contracts/order.ts`).
- **Shipping is excluded from GMV/earnings reporting** — `CurrencyTotalDto.amount` is
  documented as line GMV only, deliberately excluding `shippingTotal`
  ([[Vendor & Admin Portals]], [[Analytics & Reporting]]). Once the fee is non-zero
  this becomes a live invariant: the shipping fee must never enter a vendor's gross,
  the commission base, or `heldForVendors`. Commission is computed on `order_items`
  and shipping lives on the order header, so this holds structurally — but it must be
  covered by an explicit regression test, not left to structure.
- **Any service method touching money gets a unit test in the same PR** — non-negotiable.
- **Schema changes go through a TypeORM migration**; `synchronize` stays off. New
  non-nullable columns need a `DEFAULT` or an in-migration backfill.

Configuration pattern, mirroring [[Vendor Earnings & Commission]] VE-1:

- **Append-only effective-dated history, not a mutable value.** Changing the fee
  inserts a new row with a later `effectiveFrom`; rows are never edited or deleted.
- **New `effectiveFrom` must be strictly after the latest existing one** → 409 otherwise.
- **A unique DB index is the race gate**, not app-layer read-then-write (VE-1's
  code-review lesson: two concurrent POSTs both read the same "latest" row; catch pg
  `23505` and rethrow as the same `ConflictException`).
- **Resolution is boundary-inclusive (`effectiveFrom <= t`) and throws if no row
  covers the date** — never silently falls back to zero. A missing fee row is a bug,
  not "free shipping".
- **Admin-only, `GET` + `POST` only.** No PATCH/DELETE routes.
- **Audit-logged** via the existing `AuditService`, mirroring `commission_rate.created`.

## Data model
**No new column on `orders` is required.** `orders.shippingTotal numeric(12,2) NOT NULL
DEFAULT 0` already exists (`1781136000000-InitialSchema`), is already on `OrderDto`,
and is already the per-order money snapshot in the order's own currency — exactly the
snapshot pattern `order_items.unitPrice` uses. Writing the resolved fee into it *is*
the historical freeze. Do not add a redundant `shippingFeeId` FK or a duplicate amount
column (see Q3 if traceability to the specific config row is wanted).

New: **`shipping_fees`** — append-only, one row per (`effectiveFrom`, `originCountry`,
`destinationCountry`, `currency`) combination (Q4 resolved: route-based keying):

- `id` uuid PK
- `amount numeric(12,2)` — the flat fee, in `currency`
- `currency currency_code` — the existing shared Postgres enum
- `originCountry country_code` — the existing shared Postgres enum
- `destinationCountry country_code` — the existing shared Postgres enum
- `effectiveFrom timestamptz`
- `note varchar(500)` nullable
- `createdByUserId` uuid nullable
- `createdAt timestamptz`
- **UNIQUE (`effectiveFrom`, `originCountry`, `destinationCountry`, `currency`)** — the concurrency gate
- Index on (`originCountry`, `destinationCountry`, `currency`, `effectiveFrom`) for resolution

**A fee "set" must cover every (route, currency) combination — 4 routes × 2 currencies
= 8 rows.** One admin POST supplies amounts for all 8 combinations and inserts them in a
single transaction, so a scheduled ZAR-only change can never leave NAD orders unpriced.
`getFeeAt(date, route, currency)` takes the route as an explicit input and throws if no
row covers that combination, mirroring `CommissionRateService.getRateAt`. The seed
migration inserts a covering set for all 8 combinations at an epoch `effectiveFrom` —
Q2 resolved: seed **0.00 for every route and currency** (behaviour-neutral; nothing is
charged until an admin sets real amounts through the admin screen).

Rounding: none needed. The fee is a configured 2dp amount added whole; it is never
multiplied or apportioned. Order math stays in integer cents, matching
`OrdersService.create`'s existing `subtotalCents` handling.

## Per-product shipping fee override (added 2026-08-23)
**Use case:** HB pre-positions bulk stock of certain fast-moving products in Namibia
ahead of demand. The marginal shipping cost for a unit sold from that pre-positioned
stock is lower than the standard cross-border cost, so the admin needs to set a
cheaper (or, symmetrically, pricier) fee for specific products that beats the global
default. **The override is keyed **(productId, originCountry, destinationCountry,
currency)** — per route as well as currency.** A product with pre-positioned stock
in Namibia is cheap to ship **NA→NA** but costs the normal amount **ZA→NA**, which a
currency-only key cannot express.

**Design: mutable, not effective-dated history — deliberately unlike the global
default.** The global fee needed an append-only history because it is a single
platform-wide value where *when* a change took effect matters for every order placed
platform-wide. A per-product override is a narrow, local exception on one product,
the same shape of value as `products.price` — which is already a plain mutable
column, edited via `PATCH /products/:id`, with no history table. Mirroring
`price`'s precedent is more consistent than mirroring `commission_rates` here. As
with `price`, the value actually charged is frozen onto the order at creation time
(`orders.shippingTotal`), so historical order integrity does not depend on the
override itself being historical. **Overrides remain deliberately sparse** (a product
may override one combination and fall back to the default for the rest) and
**deliberately mutable** (no effective-dated history).

**Data model:** new `product_shipping_fee_overrides` table — `id` uuid PK,
`productId` uuid FK → `products.id` (`ON DELETE CASCADE`), `originCountry country_code`,
`destinationCountry country_code`, `currency currency_code`, `amount numeric(12,2)`,
`updatedAt timestamptz`, `updatedByUserId` uuid nullable. **UNIQUE (`productId`,
`originCountry`, `destinationCountry`, `currency`).** Unlike the global default, an
override does not need to cover every (route, currency) combination — the admin may set
an override for just **NA→NA/NAD** (the Namibia pre-positioned-stock case) and leave
all other combinations unset, in which case they fall back to the global default via
`ShippingFeeService.getFeeAt`. Admin-only upsert (set/replace the amount for one route
and currency) and delete (clear the override, reverting that combination to the global
default); no separate "history" list — the current row is the value, same as `price`.

**Resolution algorithm, run once per order at creation (extends SF-3):** for each
`order_item`, resolve that line's fee as `override[product, order.originCountry,
order.destinationCountry, order.currency] ?? ShippingFeeService.getFeeAt(orderCreatedAt,
order.originCountry, order.destinationCountry, order.currency)`. Then
`orders.shippingTotal = MAX(...)` over every line's resolved fee — **decided
2026-08-23: the highest applicable fee across the cart wins**, not a sum and not an
average. Example given: a cart with one item overridden to R50 (NA→NA) and one item at the
R250 default (ZA→NA) charges R250 total; a cart with one item overridden to R400 and one at
the R250 default charges R400 total. This keeps shipping a single order-level
amount — `orders.shippingTotal` needs no schema change, and no per-line-item
shipping charge is introduced (`order_items` is untouched). Still resolve strictly
against `order.originCountry` and `order.destinationCountry`; never mix an override
configured for one route into an order placed on a different route or currency.

**Admin UI requirement:** `admin-catalog`'s product tab **explicitly excludes vendor
listings today** — "Only platform listings — vendor listings are never shown here"
(`admin-catalog.ts:41`, by design, since vendors own editing their own listings). This
feature needs a **new, read-only "Vendor Products" tab** alongside the existing
Products/Categories tabs: lists vendor-listed products (the existing `GET /products`
endpoint already returns both listing types and already accepts `vendorId`; it has no
`listingType` filter yet, so the tab likely wants one added, or can filter
client-side the same way `platformProducts` does today, inverted). The admin can view
each vendor product (name, vendor, price, current effective shipping fee) and open a
small control to set/clear its override per (route, currency) combination — not the
full create/edit product form, since the admin does not own vendor product content.

## @hb/shared contract impact
New `libs/shared/src/contracts/shipping-fee.ts` (interfaces + enums only; reuse
`CurrencyCode` and `CountryCode` — do not duplicate). Note `contracts/shipping.ts` already exists for the
`SHIPPING_PROVIDER` courier port; keep the fee contract separate from courier concerns.

- **`ShippingFeeDto`** — `id`, `amount`, `currency`, `originCountry`, `destinationCountry`,
  `effectiveFrom`, `note?`, `createdAt`, `createdByUserId?`. **Route-keyed** (Q4 resolved).
  Mirrors `CommissionRateDto`.
- **`ShippingFeeSetDto`** — `effectiveFrom`, `fees: ShippingFeeDto[]` (one entry per route/currency
  combination — 8 total), `inForce: boolean`. The history list groups by `effectiveFrom` so the admin
  sees one row per change, not one per combination.
- **`ShippingFeeHistoryDto`** — `{ items: ShippingFeeSetDto[] }`, newest first.
- **`CreateShippingFeeSetEntry`** — `originCountry`, `destinationCountry`, `currency`, `amount`.
- **`CreateShippingFeeSetRequest`** — `fees: CreateShippingFeeSetEntry[]` (must cover all 8 route/currency
  combinations), `effectiveFrom?` (ISO-8601, defaults to now), `note?`.
- **`CurrentShippingFeeDto`** — `{ amount, currency, originCountry, destinationCountry }`. Read model for
  the checkout preview, so the UI can display the fee before the order exists. Checkout specifies destination
  explicitly; `originCountry` is optional and derived server-side from the cart (using the same resolution
  rule as `OrdersService.create`) to guarantee the previewed fee matches the fee actually charged.
- **`GetCurrentShippingFeeQuery`** — `originCountry?: CountryCode`, `destinationCountry: CountryCode`,
  `currency: CurrencyCode`.
- **`OrderDto.shippingTotal`** — unchanged; its doc-comment should stop implying zero.
- **`ProductShippingFeeOverrideRoute`** — `originCountry`, `destinationCountry`, `currency`.
- **`ProductShippingFeeOverrideDto`** (added 2026-08-23) — extends `ProductShippingFeeOverrideRoute`;
  adds `id`, `productId`, `amount`, `updatedAt`, `updatedByUserId?`. **Route-keyed override** (Q4 resolved).
- **`SetProductShippingFeeOverrideRequest`** — extends `ProductShippingFeeOverrideRoute`; adds `amount`
  (non-negative, ≤2dp, same validation as the global fee).
- **`ProductDto`** gains an optional `shippingFeeOverrides?: ProductShippingFeeOverrideDto[]`
  (empty/absent ⇒ product uses the global default for every route/currency combination) so the admin
  vendor-products tab can render current overrides without a second round-trip.

Every endpoint input is a class-validator DTO implementing the shared interface.

## Out of scope (recorded so nobody assumes them)

- **Any variable fee** — per weight, per volumetric size, per vendor, per region,
  per distance, per shipment. v1 ships one global flat fee per currency (Q4), plus
  the per-product override above — nothing finer-grained (no per-shipment, no
  quantity-scaled override).
- **Per-line-item shipping charges.** The override changes which single fee wins for
  the whole order (highest applicable, decided 2026-08-23); it does not make shipping
  a per-item charge or add a fee to `order_items`. One order still shows one shipping
  line.
- **Vendor-initiated overrides.** Only the admin can set a product's override, even on
  a vendor-owned listing — this is an admin pricing lever, not a vendor one.
- **Full admin edit rights over vendor product content** via the new Vendor Products
  tab. That tab is read-only except for the shipping-fee-override control.
- **Free-shipping thresholds, promo codes, discounts.** No vendor-run promotions exist
  at Phase 1 launch ([[HB Domain Model]] pricing resolution).
- **Priority/expedited delivery flat fee** — listed as TBD upstream in
  `H&B Brain/08-revenue-model.md`; a separate fee, not this one.
- **Splitting the fee per vendor, or charging it per shipment** when an order spans
  multiple vendors. One order, one fee.
- **Wiring `SHIPPING_PROVIDER` to book a real shipment** ([[Order State Machine]]
  follow-up, [[Cross-Border & Customs]]) — the port stays a stub. This note prices a
  fee; it does not buy a courier label.
- **Reconciling the fee against actual courier cost**, or any margin/variance reporting.
- **Customs duties.** SACU means no duty today ([[Cross-Border & Customs]]); a duty
  line item is not this fee.
- **Refunding the shipping fee** on cancellation — no refund flow exists in code
  ([[Vendor Earnings & Commission]] out-of-scope), so there is nothing to hook.
- **Backfilling historical orders.** Past orders keep `shippingTotal = 0.00`; that is
  what was actually charged and it must stay true.
- **Changing GMV/earnings semantics.** Shipping stays excluded from `CurrencyTotalDto`.

## Open questions (a human must answer Q1–Q2 before SF-1 starts)
## Resolved questions (2026-08-24)

1. **Flat fee vs pass-through-at-cost — RESOLVED.** [[HB Domain Model]] and `H&B Brain/08-revenue-model.md`
   record the delivery-fee model as settled at "pure pass-through at cost, no margin". The flat fee is not
   pass-through. **Resolution:** the flat fee is an **operational approximation of average pass-through cost**,
   which is compatible with the stated business model — the business model is NOT changing. The note has
   been updated (see Problem section above) to reflect this. **Action item (outstanding, human review):**
   The live marketing copy in [[Landing Site Migration]] states "shipping + customs passed through". This
   is now flagged for human review — it must be updated or retracted depending on business intent. The
   feature shipped without changing this copy; it must not be silently left as-is. **Date resolved:** 2026-08-24.

2. **Seed amounts — RESOLVED.** **Resolution:** seed **0.00 for every (route, currency) combination**
   (8 rows total). The migration is behaviour-neutral; nothing is charged until an admin sets real amounts
   through the admin screen. Every combination (ZA→ZA, ZA→NA, NA→NA, NA→ZA, each in ZAR and NAD) starts
   at R0.00 / N$0.00. **Date resolved:** 2026-08-24.

3. **Traceability of the applied fee.** `orders.shippingTotal` records the amount charged, which is enough
   to keep every historical total correct. No FK or history table for the config row was implemented — the
   value snapshot is the only record. This precedent matches `commission` (which is a percentage, not money)
   and remains acceptable. **Not blocking; left as-is.**

4. **Route variance — RESOLVED.** **Resolution:** the fee is keyed on **(route, currency), not currency alone**.
   A route is an origin→destination `CountryCode` pair; `CountryCode` has exactly two members (ZA, NA), so
   there are exactly **4 routes — ZA→ZA, ZA→NA, NA→NA, NA→ZA — all configurable**. This is now a **scope
   change the note carries as resolved:** a fee "set" is **8 rows (4 routes × 2 currencies), all required,
   one transaction**. NA→ZA is included deliberately: `orders.originCountry` is derived from
   `product.originCountry` and can legitimately be NA, so that order shape is representable and must never
   resolve to a guessed fee. The checkout label "Shipping (ZA to NA)" has been replaced with a label built
   from the resolved route on the returned DTO, so it can never assert a route that isn't being charged.
   **Date resolved:** 2026-08-24.

5. **Minimum-order interaction — RESOLVED.** **Resolution:** confirmed, no interaction. A R50 order carries
   the same fee as a R5,000 order. No free-shipping threshold, no scaling. **Date resolved:** 2026-08-24.

## Implementation Notes (2026-08-24)

**What shipped:** Six cards bundled into one PR: SF-1 (shipping_fees table, service, admin history/GET/POST),
SF-2 (admin shipping-fee screen), SF-3 (fee resolution at order creation, GET /shipping-fee/current for
checkout), SF-4 (checkout line item rendering), SF-5 (product_shipping_fee_overrides table, admin override
endpoints), SF-6 (admin Vendor Products tab with override controls).

**Key decisions:**
- Route is (originCountry, destinationCountry), not just the destination. This allows pre-positioned stock
  scenarios (e.g. NA→NA cheap, ZA→NA standard) and covers all representable order shapes.
- Fee "set" is all 8 (route, currency) combinations in one atomic POST, no partial updates. One transaction
  gates concurrency via UNIQUE.
- Per-product override is mutable (not effective-dated), mirroring `products.price`, because the value
  actually charged is frozen onto the order at creation time.
- MAX across cart lines (not sum, not average) — one order, one shipping line, determined by the highest
  override or default applicable to any line in that order.
- Checkout fee resolution uses the same helper (`resolveShippingCents`) as order creation, with a parity
  spec ensuring preview matches actual charge.

**Code review findings (fixed in this batch):** checkout preview initially returned global default only while
order creation charged `MAX(override ?? default)` — a cart with a higher override would display one total and
authorize a larger one, with the real figure appearing only on confirmation (after charge). Fixed by extracting
`resolveShippingCents` into a shared helper called by both code paths, plus parity spec. Also fixed: spec
asserting stubbed value against itself; order placement possible while fee unknown; missing `ParseUUIDPipe`
on override endpoints (malformed ID hit DB, returned 500); admin history rendering absent (route, currency)
as R0.00 (read as free shipping).

**Verification:** `npm run lint:api` clean, `npm run test:api` 931 passed, `npm run test -w @hb/web` 963
passed, `npm run build` exit 0. Migrations verified up → down → up against live dev Postgres; all 8 seed rows
confirmed present at 0.00.

**Follow-ups (not shipped, flagged for later):**
- Marketing copy review: "shipping + customs passed through" wording in [[Landing Site Migration]] needs
  human decision to update or retract.
- Checkout banner: top "Cross-Border Shipping" banner still hardcodes "South Africa to Namibia" (separate
  from this feature, being handled separately).
- Vendor Products tab at scale: currently issues one override GET per vendor product row on tab open (bounded
  by 100-item admin catalog cap, but would want server-side support if vendor catalog grows).

**PR/Card:** feat/DwnyCLnX-configurable-shipping-fee (bundled batch of 6 cards: SF-1 #126 DwnyCLnX,
SF-5 #130 EjWUrO6V, SF-3 #128 aQTPJZJK, SF-2 #127 ybWTncOf, SF-4 #129 BU7PoQIm, SF-6 #131 eTv2Guoq).