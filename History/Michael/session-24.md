# Session 24 — VE-3: Payout-eligibility data model & net earnings computation

**Date:** 2026-08-03  
**Card:** VE-3 (#72, shortLink `trMZD1C5`, board "H&B E-commerce")  
**Feature:** Vendor Earnings & Commission (backend engine, third slice of 5)  
**Branch:** `feat/trMZD1C5-payout-eligibility-earnings`  
**Status:** PR open, not yet merged

## What was built

The backend engine layer for payout eligibility and net earnings computation: `deliveredAt` timestamp column on orders + commission-rate snapshot on order items + a standalone `VendorEarningsService` that computes eligibility, claim-window expiry, accruing balance, and settlement-period bucketing. This card ships the data model and service logic (no HTTP surface or UI — those are VE-4 and VE-5). It depends on VE-1 (rate history) and directly unblocks VE-4 and VE-5.

**Single subagent — API-only card, no UI changes.**

## Changes

**Repo:** hb-mono-repo  
**Branch:** `feat/trMZD1C5-payout-eligibility-earnings`

### 1. Migrations

**`1784505600000-OrderDeliveredAt`**
- Adds `orders.deliveredAt timestamptz NULL` with index.
- Backfills existing `delivered` status rows from `updatedAt`.
- Symmetric down-migration.

**`1784592000000-OrderItemCommissionRatePercent`**
- Adds `order_items.commissionRatePercent numeric(5,2) NULL`.
- Backfill resolves each order's actual effective-dated rate via correlated subquery (mirrors `CommissionRateService.getRateAt` — latest `commission_rates` row with `effectiveFrom <= order.createdAt`).
- Platform lines (`vendorId IS NULL`) backfill to NULL.

### 2. OrdersService updates

**`OrdersService.updateStatus`**
- Stamps `deliveredAt` exactly once on `shipped → delivered` via atomic conditional `UPDATE ... WHERE deliveredAt IS NULL` (prevents race-condition double-stamp).

**`OrdersService.create`**
- Snapshots `commissionRatePercent` per vendor line from `CommissionRateService.getRateAt()`, resolved once per order (single call across all lines).
- Platform lines receive `null`.
- Added `CommissionModule` to `OrdersModule`'s imports (per VE-1: `CommissionModule` is not `@Global()`).

### 3. New earnings module (`apps/api/src/earnings/`)

**Backend-only engine (no HTTP surface):**
- `earnings.constants.ts` — `DAMAGE_CLAIM_WINDOW_HOURS = 48`, `SETTLEMENT_ANCHOR_DATE` placeholder.
- `vendor-earnings.service.ts` — `VendorEarningsService.getEarnings(scope, from, to, now?)` returning `{ pendingClaimWindow, accrued, settlementPreview }`.

### 4. Tests

**12 new/extended unit tests:**
- `orders.service.spec.ts` — create/update stamp logic, rate snapshot per-order consistency.
- `vendor-earnings.service.spec.ts` — claim-window boundary (inclusive at 48h), cancellation/refund exclusion, rounding drift, ZAR/NAD separation, vendor scoping, settlement-anchor bucketing.

## Key decisions worth recording

### 1. `VendorEarningsService` method signature without shared contracts

**What:** `getEarnings(scope: { vendorId?: string }, from: Date, to: Date, now = new Date())` — no `@hb/shared` DTOs, shape chosen because VE-4/VE-5 build their own endpoint contracts on top.

**Why:** This card is the engine; the HTTP/UI layers (VE-4 cross-vendor admin, VE-5 vendor own-earnings) will each define their own request/response DTOs. Locking in shared contracts now would either duplicate them or force a premature DTO design.

**How:** `scope.vendorId` undefined = platform-wide (VE-4 use case); concrete id = scoped to one vendor (VE-5 use case) with ownership check in service layer.

**Impact:** VE-4 and VE-5 can independently design their public API contracts without rework.

### 2. Settlement periods bucket on eligibility instant, not delivery instant

**What:** `periodIndexFor` is keyed off `deliveredAt + 48h`, not `deliveredAt` itself.

**Why:** A line delivered in the last 48h of a bi-weekly period doesn't become payout-eligible until the next period. Bucketing on `deliveredAt` would misplace it in a period a settlement batch could never have paid it in.

**How:** `eligibilityInstant = deliveredAt + 48h`; `periodIndex = floor((eligibilityInstant - SETTLEMENT_ANCHOR_DATE) / 14 days)`.

**Impact:** Settlement-period reports accurately reflect which lines were actually eligible in each batch.

### 3. Rounding in integer "hundredths-of-a-percent" units

**What:** Convert `ratePercent` (e.g., 8.29) to integer (829 hundredths) before any division.

**Why:** `Math.round(gross * 8.29 / 100)` misrounds at exact-half-cent boundaries due to IEEE 754 float arithmetic (8.29 * 100 = 828.9999999...). Leads to silently wrong commission (e.g., R50.00 @ 8.29% rounds down to 414 instead of 415).

**How:** `const rateHundredths = Math.round(ratePercent * 100); commission = Math.round(gross * rateHundredths / 10000)`.

**Impact:** Commission calculations are exact to the cent; no silent under-payments to vendors or over-payments to H&B.

### 4. Null commission rate is an exclusion, not a 0% default

**What:** If `commissionRatePercent IS NULL`, the line is excluded from all earnings buckets (with a warning logged).

**Why:** Should be unreachable post-migration, but was an untested code path. Silently treating `NULL` as 0% commission is a silent-revenue-loss risk if a backfill gap or bad row ever occurs.

**How:** Service explicitly checks `if (line.commissionRatePercent === null) { logger.warn(...); continue; }`.

**Impact:** Any data anomaly is visible in logs; revenue is never silently under-calculated.

### 5. Backfill resolves actual historical rate per order

**What:** Migration-2 backfill uses a correlated subquery: for each `order_item`, find the latest `commission_rates` row with `effectiveFrom <= order.createdAt`.

**Why:** VE-1 ships effective-dated history; VE-2 ships admin UI to add new rates. A flat "everyone gets 15.00%" backfill would silently misstate any line whose order predates a later rate change (e.g., orders placed before a 15% → 12% reduction would show 15% instead of the 12% that actually applied at their creation).

**How:** 
```sql
UPDATE order_items oi
SET commissionRatePercent = (
  SELECT cr.ratePercent FROM commission_rates cr
  WHERE cr.effectiveFrom <= o.createdAt
  ORDER BY cr.effectiveFrom DESC LIMIT 1
)
FROM orders o WHERE o.id = oi.orderId AND oi.vendorId IS NOT NULL
```

**Impact:** Historical earnings reports are accurate even when rates have changed.

## Review & test outcome

**Code review pass** found 4 blocking issues, all fixed before PR:

1. **Rounding float-precision bug** — `Math.round(gross * ratePercent / 100)` misrounds 8.29% cases. Fixed: convert rate to integer hundredths before division. Test verifies R50.00 @ 8.29% = R4.15 exactly.

2. **Hardcoded-rate backfill** — initial backfill used "everyone gets 15.00%". Fixed: backfill now resolves actual historical rate per order via correlated subquery.

3. **Eligibility-instant bucketing off-by-one** — `periodIndexFor` was keyed on `deliveredAt`. Fixed: key off `deliveredAt + 48h` instead. Test verifies claim-window-end line goes to next period.

4. **Untested null-rate branch** — code path treating `commissionRatePercent IS NULL` as 0% was unreached. Fixed: now explicitly excluded with warning log. Test verifies null-rate line is skipped and warning is emitted.

**Non-blocking fix applied proactively:**
- `deliveredAt` stamp race — two concurrent `shipped → delivered` calls via whole-entity save. Fixed via atomic conditional DB update (`WHERE deliveredAt IS NULL`). Test verifies only one stamp succeeds.

**Test results:**
- `npm run test:api`: 535/535 pass (39 suites).
- `npm run lint:api`: clean.
- `npm run build` (root): clean (shared → api → web).
- Migrations verified up → down → up against existing non-empty dev DB (twice: before and after backfill-query fix).

## What's not in scope (out-of-scope clarifications for this card)

- Any endpoint, HTTP controller, or `@hb/shared` contract (VE-4/VE-5 build those).
- Any UI or vendor-facing portal (VE-5).
- Executing payouts or settlement job (separate TBD, out of scope entirely).
- Cancellation-fee amount (separate TBD upstream).
- Minimum payout amount, payment gateway integration (separate TBDs).

## Follow-ups

### Next in chain (unblocked by this card):

- **VE-4** (#73, `1DAb9a1I`) — Admin cross-vendor earnings: `GET /admin/earnings` endpoint + UI. **Depends on VE-3** for `VendorEarningsService.getEarnings()`. Unblocked by this card.
- **VE-5** (#74, `WrPOgQDa`) — Vendor own-earnings: `GET /vendors/me/earnings` endpoint + portal screen. **Depends on VE-3**. Unblocked by this card.

### No open items from this card:
- Commission rounding precision documented (pattern used in VE-2 UI, now mirrored in VE-3 backend).
- Historical rate resolution in backfill ensures accurate past earnings.
- Null-rate exclusion is logged and tested.
- Atomic `deliveredAt` stamp prevents race conditions.

## PR link

Trello card 72: https://trello.com/c/trMZD1C5  
Branch: `feat/trMZD1C5-payout-eligibility-earnings`  
PR: (link pending, PR open but not yet merged as of session date)
