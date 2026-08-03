# Session 22 — VE-1: Admin-configurable commission rate

**Date:** 2026-08-03  
**Card:** VE-1 (#70, shortLink `QAeB8YGv`, board "H&B E-commerce")  
**Feature:** Vendor Earnings & Commission (foundation slice of 5)  
**Branch:** `feat/QAeB8YGv-admin-commission-rate`  
**Status:** PR open, not yet merged

## What was built

The foundation layer for the "Vendor Earnings & Commission" feature (VE-1 → (VE-2 ∥ VE-3) → (VE-4 ∥ VE-5)): admin-configurable commission rate with effective-dated history, ensuring past earnings are never retroactively restated.

**Business rule implemented** (from `H&B Brain/08-revenue-model.md`, section 08-revenue-model): platform takes 15% commission on vendor order lines; vendor keeps 85%. The rate is provisional and must be admin-configurable, so it's append-only: the applicable rate at any moment = the row with the greatest `effectiveFrom <= t`.

## Changes

**Repo:** hb-mono-repo  
**Branch:** `feat/QAeB8YGv-admin-commission-rate`

### 1. Shared contracts (`libs/shared/src/contracts/earnings.ts`)

**New file:**
- `CommissionRateDto` — read model for admin rate-history list (id, ratePercent, effectiveFrom, createdAt).
- `CreateCommissionRateRequest` — admin endpoint input (percent, effectiveFrom, optional note).
- `CommissionRateListItemDto` — extends `CommissionRateDto` with `inForce: boolean` flag to mark the currently-active rate in the list.
- `CommissionRateListDto` — wrapper for paginated list response.

All exported from `contracts/index.ts`.

### 2. API module (`apps/api/src/commission/`)

**New module structure:**
- `commission.entity.ts` — `CommissionRateEntity` mapped to `commission_rates` table.
  - Columns: id (PK), ratePercent (numeric(5,2)), effectiveFrom (timestamptz), note (varchar(500)), createdByUserId (FK to users.id), createdAt (timestamptz).
  - **Unique index on `effectiveFrom`** (critical for race prevention; see "Key decisions" below).
  
- `commission.service.ts` — `CommissionRateService`.
  - `create(req: CreateCommissionRateRequest, userId: string)` — validates `effectiveFrom` is strictly after the latest existing row (409 Conflict if not), then inserts. Catches pg `23505` unique-constraint violations and rethrows as `ConflictException`. Logs audit action `commission_rate.created` via `AuditService`.
  - `list()` — returns all rows ordered by `effectiveFrom DESC`, computing `inForce` flag on the top row (current rate).
  - `getRateAt(date: Date)` — returns the row with the greatest `effectiveFrom <= date`. Used by downstream cards (VE-3) to snapshot the rate that applied at order-creation time. Throws if no row covers the date (should never happen post-seed, but VE-3 must not treat this as "no commission").

- `commission.controller.ts` — HTTP surface for admin rate management.
  - `GET /admin/commission-rates` (no query params for v1, all history) → `CommissionRateListDto`.
  - `POST /admin/commission-rates` (body: `CreateCommissionRateRequest`) → `CommissionRateDto`.
  - Both routes guarded by `@Roles(ADMIN)`.
  - No PATCH/DELETE routes (append-only by omission).

- `commission.module.ts` — imports TypeORM, exports `CommissionRateService` but NOT `@Global()` (see "Key decisions" below).

### 3. Database migration (`apps/api/src/database/migrations/1784419200000-CommissionRates.ts`)

Creates `commission_rates` table:
```sql
CREATE TABLE commission_rates (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  rate_percent numeric(5,2) NOT NULL,
  effective_from timestamptz NOT NULL,
  note varchar(500),
  created_by_user_id uuid NOT NULL REFERENCES users(id),
  created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(effective_from)
);
```

**Seed logic:** inserts the 15.00% starting rate with `effectiveFrom = (earliest order.createdAt OR epoch)`. Ensures every date from day one has a defined rate for `getRateAt()` queries.

### 4. Tests

**14 unit tests** (`commission.service.spec.ts` + `commission.controller.spec.ts`):

**Service tests:**
- **Strictly-after boundary** — rejects `effectiveFrom <= latest.effectiveFrom` with 409; accepts strictly-later dates.
- **Duplicate insertion** — verifies that pg `23505` unique-constraint violation is caught and rethrown as `ConflictException`.
- **Numeric string coercion** — POST body `{ percent: "15.5" }` correctly coerces to 15.5 (number).
- **`inForce` flagging** — list response marks only the top row (highest `effectiveFrom`) as `inForce: true`.
- **`getRateAt` boundary resolution** — asserts actual `LessThanOrEqual` query logic (not just mocked return).
  - Covers exact date match, date between rows, and date before any row (throws).

**Controller tests:**
- Admin-only access guard (non-admin requests receive 403).
- Route registration in the public-route auth guardrail test.

### 5. Audit trail

New audit action: `commission_rate.created` logged whenever a rate is inserted, via existing `AuditService` integration.

## Key decisions for downstream cards

### 1. Race prevention via unique index, not app logic

**What:** Unique constraint on `effectiveFrom` in the migration.

**Why:** Initial implementation had only app-layer read-then-write logic. Code review caught a race: two concurrent POSTs could both read the same "latest" row and proceed to insert, violating the "strictly-after" invariant if they assigned the same `effectiveFrom`. The database, not the app, must be the race-prevention gate.

**How:** Add `UNIQUE(effective_from)` to the migration. Service catches pg `23505` violations and rethrows as `ConflictException`. Sequential attempts (if one succeeds while another is in flight) get the same error, so the client-side retry/error handling is uniform.

**Follow-up:** If rate insertions become concurrent at scale (unlikely, but admin actions are usually low-frequency), consider adding an advisory lock (`SELECT pg_advisory_lock()` on a fixed value before read) if the business demands strict linearization. For now, the unique index is sufficient.

### 2. `CommissionModule` is NOT `@Global()`

**What:** `commission.module.ts` does not carry `@Global()`.

**Why:** Initially built `@Global()` to mirror `AuditModule`'s pattern. But code review flagged: `CommissionModule` owns an HTTP controller (audit does not) and currently has no cross-module consumer. Globalizing it would export HTTP surface into every module's injector without reason yet.

**How:** VE-3 should explicitly `imports: [CommissionModule]` when it needs `CommissionRateService.getRateAt()` for order-item rate snapshots. This keeps the module boundary explicit and prevents accidental coupling.

**Impact on VE-3:** Don't assume `CommissionRateService` is injectable everywhere. Add it to the `imports` array in `OrdersModule` or wherever `OrdersService` lives.

### 3. `getRateAt(date)` throws if no row covers the date

**What:** No silent fallback; throws if `date < earliest effectiveFrom`.

**Why:** Should never happen post-seed (seed inserts the 15% rate at epoch or earliest order time). But if it does, VE-3 (which uses this to snapshot the rate for an order-item) must not silently treat it as "no commission" — that would silently produce zero-commission lines, corrupting the earnings model.

**How:** VE-3 must catch/log this error explicitly and decide: is it a data integrity issue (propagate to client as 500) or a valid edge case (e.g., order created in a migration window before the seed ran)? Current spec assumes it won't happen; revisit if it does.

## Review outcome

**Code review pass** found 4 blocking issues, all fixed before PR:

1. **Weak `getRateAt` test assertions** — test was mocking the return but not verifying the query operator. Fixed: now asserts the actual `LessThanOrEqual` behavior.
2. **Missing unique constraint** — added to migration to prevent concurrent-insert race.
3. **Unbounded `note` length** — arbitrary varchar could balloon row size. Fixed: limited to 500 chars via migration `varchar(500)`.
4. **`CommissionController` not in public-route auth guardrail test** — guard test wasn't registering the new controller. Fixed: added controller to the test setup.

**Test suite:**
- `npm run test:api`: 512/512 pass (existing suite + 14 new tests).
- `npm run lint:api`: clean (eslint + prettier).
- `npm run build`: clean (shared → api → web).

## What's not in scope (out-of-scope clarifications for this card)

- `orders`, `order_items` changes (VE-3).
- Payout eligibility, claim windows, settlement (VE-3/VE-4/VE-5).
- Admin UI for rate management (VE-2).
- Per-vendor negotiated rates (Phase 1.5).

## Follow-ups

### Next in chain (unblocked by this card):

- **VE-2** (#71, `WIZ4pJtk`) — Admin portal commission-rate management screen. Can start any time; frontend only.
- **VE-3** (#72, `trMZD1C5`) — Payout-eligibility data model (`deliveredAt` + claim-window) + net earnings computation. **Depends on VE-1** for `CommissionRateService.getRateAt()`. Must `imports: [CommissionModule]` (see decision 2 above). Uses `getRateAt()` to snapshot the rate that applied at order-creation time.

### Outstanding decisions (spec-level, not blocking this card):

- `deliveredAt` granularity — order-level (v1 default) vs per-shipment. Spec defers to VE-3, revisit only if partial delivery is itself specced.

## PR link

Trello card 70: https://trello.com/c/QAeB8YGv  
Branch: `feat/QAeB8YGv-admin-commission-rate`  
(PR pending merge, not yet linked in this session)
