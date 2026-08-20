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

**Unresolved upstream conflict — see Open Questions Q1.** [[HB Domain Model]] records
the delivery-fee *model* as settled at "pure pass-through at cost, no margin"
(`H&B Brain/08-revenue-model.md`), and [[Landing Site Migration]] carries
"shipping + customs passed through" into public marketing copy. A single global flat
fee is not pass-through-at-cost. This spec is written against the request as stated,
but implementation must not start until Q1 is answered by a human.

## Scope

In scope: one global flat shipping fee per currency, admin-configurable via an
effective-dated history; applied at order creation; surfaced as a distinct line item
at checkout and on the order; frozen onto the order as charged.

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

New: **`shipping_fees`** — append-only, one row per (`effectiveFrom`, `currency`) pair:

- `id` uuid PK
- `amount numeric(12,2)` — the flat fee, in `currency`
- `currency currency_code` — the existing shared Postgres enum
- `effectiveFrom timestamptz`
- `note varchar(500)` nullable
- `createdByUserId` uuid nullable
- `createdAt timestamptz`
- **UNIQUE (`effectiveFrom`, `currency`)** — the concurrency gate
- Index on (`currency`, `effectiveFrom`) for resolution

**A fee "set" must cover every `CurrencyCode`.** One admin POST supplies an amount for
each currency and inserts all rows in a single transaction, so a scheduled ZAR-only
change can never leave NAD orders unpriced. `getFeeAt(date, currency)` throws if no row
covers that pair, mirroring `CommissionRateService.getRateAt`. The seed migration
inserts a covering set for both currencies at an epoch `effectiveFrom` — Q2 decides the
seed amounts (0.00 is the safe default: identical to today's behaviour, so the
migration is behaviour-neutral until an admin sets a real fee).

Rounding: none needed. The fee is a configured 2dp amount added whole; it is never
multiplied or apportioned. Order math stays in integer cents, matching
`OrdersService.create`'s existing `subtotalCents` handling.

## @hb/shared contract impact

New `libs/shared/src/contracts/shipping-fee.ts` (interfaces + enums only; reuse
`CurrencyCode` — do not duplicate). Note `contracts/shipping.ts` already exists for the
`SHIPPING_PROVIDER` courier port; keep the fee contract separate from courier concerns.

- **`ShippingFeeDto`** — `id`, `amount`, `currency`, `effectiveFrom`, `note?`,
  `createdAt`, `createdByUserId?`. Mirrors `CommissionRateDto`.
- **`ShippingFeeSetDto`** — `effectiveFrom`, `fees: ShippingFeeDto[]` (one per currency),
  `inForce: boolean`. The history list groups by `effectiveFrom` so the admin sees one
  row per change, not one per currency.
- **`ShippingFeeHistoryDto`** — `{ items: ShippingFeeSetDto[] }`, newest first.
- **`CreateShippingFeeSetRequest`** — `fees: { currency: CurrencyCode; amount: number }[]`
  (must cover every currency), `effectiveFrom?` (ISO-8601, defaults to now), `note?`.
- **`CurrentShippingFeeDto`** — `{ amount, currency }`. Read model for the checkout
  preview, so the UI can display the fee before the order exists.
- **`OrderDto.shippingTotal`** — unchanged; its doc-comment should stop implying zero.

Every endpoint input is a class-validator DTO implementing the shared interface.

## Out of scope (recorded so nobody assumes them)

- **Any variable fee** — per weight, per volumetric size, per vendor, per region,
  per distance, per shipment. v1 ships one global flat fee per currency (Q4).
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

1. **Flat fee vs pass-through-at-cost — a direct contradiction with recorded strategy.**
   [[HB Domain Model]] records the delivery-fee model as settled: "pure pass-through at
   cost, no margin" (`H&B Brain/08-revenue-model.md`), and [[Landing Site Migration]]
   put "shipping + customs passed through" into **live public marketing copy**. A fixed
   fee on every order is not pass-through. Is this a deliberate change of business
   model (→ update `H&B Brain/08-revenue-model.md`, [[HB Domain Model]], and the
   Services page copy), or is the flat fee meant as an operational approximation of
   average cost (→ acceptable, but the marketing wording still needs review)?
   **Blocking.**
2. **Seed amounts.** What are the launch ZAR and NAD amounts? The migration seeds
   `0.00` for both by default (behaviour-neutral) unless real numbers are supplied.
   Related: is the NAD amount numerically equal to the ZAR one? Even under the 1:1 peg
   it must be stored explicitly, never derived.
3. **Traceability of the applied fee.** `orders.shippingTotal` records the amount
   charged, which is enough to keep every historical total correct. Does the business
   also need to know *which* config row produced it (a `shippingFeeId` FK)? Default:
   no — commission set the precedent of snapshotting the value, not the row.
4. **Does a flat global fee need to vary later** — by destination country, weight, or
   vendor? v1 is scoped to a single global flat fee per currency. Confirm that a
   ZA→ZA **domestic** order should be charged the identical fee as a ZA→NA
   cross-border order. `orders.destinationCountry` can legitimately be `ZA`, and the
   existing checkout label reads "Shipping (ZA to NA)", which is already wrong for
   domestic orders and gets more wrong once the number is non-zero.
5. **Minimum-order interaction.** The fee applies to every order regardless of size,
   so a R50 order carries the same fee as a R5,000 one. Confirm that is intended.
