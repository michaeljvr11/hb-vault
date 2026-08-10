# Session 25 — VE-4: Admin cross-vendor earnings report

**Date:** 2026-08-10  
**Card:** VE-4 (#73, shortLink `1DAb9a1I`, board "H&B E-commerce")  
**Feature:** Vendor Earnings & Commission (admin reporting, fourth slice of 5)  
**Branch:** `feat/1DAb9a1I-admin-earnings-report`  
**Status:** PR open, not yet merged

## What was built

The admin-facing cross-vendor earnings report: `GET /admin/earnings` endpoint + UI dashboard showing platform commission earned, vendor funds held for settlement, platform-listing GMV, and a per-vendor earnings table (gross/commission/net breakdown, eligible lines only). Depends on VE-1 (rate history) and VE-3 (eligibility data model + earnings service). VE-5 (vendor own-earnings) is a parallel sibling depending on VE-3, not on this card — but can opportunistically reuse the new `resolveEarningsWindow` utility.

**Two-agent card — backend (API service + endpoint) + frontend (admin dashboard component).**

## Changes

**Repo:** hb-mono-repo  
**Branch:** `feat/1DAb9a1I-admin-earnings-report`

### 1. Shared contracts (`libs/shared/src/contracts/earnings.ts` additions)

- `EARNINGS_WINDOWS` const array (`['1w', '2w', '1m']`) — single source of truth for the window enum, used by validator `@IsIn(EARNINGS_WINDOWS)` instead of duplicating the literal union.
- `AdminEarningsQuery` DTO — optional `window: EarningsWindow`, optional `vendorId`, optional `from`/`to` date range.
- `VendorEarningsSummaryDto` — per-vendor row: `vendorId`, `businessName`, `orderCount`, and `CurrencyTotalDto[]` for `grossByCurrency`, `commissionByCurrency`, `netByCurrency` (eligible-lines-only scope: `accrued` + `settlementPreview` buckets, excluding `pendingClaimWindow`).
- `AdminEarningsReportDto` — `from`, `to`, `vendors: VendorEarningsSummaryDto[]`, plus headline figures `platformCommissionByCurrency`, `platformListingGmvByCurrency`, `heldForVendorsByCurrency` (each a `CurrencyTotalDto[]`).
- Corrected `AdminDashboardDto` doc-comments on `platformRevenue`/`vendorRevenue` to point to new earnings report.

### 2. API service layer

**`apps/api/src/common/utils/earnings-window.utils.ts`** (new)
- `resolveEarningsWindow(query: AdminEarningsQuery, now = new Date())` — converts the query's window/from/to to a date range.
- `'1w'`/`'2w'` = trailing periods from now; `'1m'` = calendar month (`1st → now`).
- Explicit `from`/`to` override window and must be supplied together (throws if only one present).

**`apps/api/src/earnings/vendor-earnings.service.ts`** (extension)
- Added `getEarningsByVendor(scope: { vendorId?: string }, from: Date, to: Date, now?: Date)` — grouped-by-vendor path alongside existing `getEarnings()` (left untouched).
- Exposes `commissionAmount` alongside `netAmount` at every bucket (VE-3 only exposed net).

**`apps/api/src/admin/admin-earnings.service.ts`** (new)
- `AdminEarningsService.getReport(query: AdminEarningsQuery)` — assembles headline figures (platform commission across all vendors including inactive ones; platform GMV from platform-owned lines only; held-for-vendors snapshot as-of-now), calls `VendorEarningsService.getEarningsByVendor()` for per-vendor rows, returns `AdminEarningsReportDto`.
- `@Roles(ADMIN)` inherited from `AdminController`.

**`apps/api/src/admin/admin.controller.ts`** (extension)
- New `GET /admin/earnings` endpoint: `@Query() query: AdminEarningsQuery` → `AdminEarningsService.getReport()`.

### 3. Web UI

**`apps/web/src/app/features/admin/pages/admin-earnings/`** (new standalone component)
- Renders three headline-figure cards (platform commission, held for vendors, platform GMV).
- Tab group: "Last week" / "Last 2 weeks" / "Last month" with the "Last month" label derived from `report().from` using explicit `timeZone: 'UTC'` (SSR/hydration fix).
- Table of vendor rows (columns: business name, order count, gross, commission, net) — not sortable in this slice; each money cell stacks one line per currency, never summed.
- No `vendorId` filter control in the UI yet (API supports it; this slice only wires the window tabs).
- Error states, loading spinner, empty states for zero vendors / zero-activity vendor rows.

**Routes & navigation**
- New `/admin/earnings` lazy-loaded route in `apps/web/src/app/app.routes.ts`.
- "Earnings Report" sidebar nav entry in admin shell pointing to `/admin/earnings`.

### 4. Tests

**API**
- `admin-earnings.service.spec.ts` (new):
  - Headline figures computation (platform commission sums all vendors including inactive; platform GMV is platform-owned only, not delivered-gated).
  - Per-vendor rows built from eligible buckets only, `pendingClaimWindow` excluded.
  - `vendorId` filter narrows everything, including the headline figures.
  - Cross-vendor reconciliation (gross = commission + net) and ZAR/NAD-never-summed.
  - Query-builder bind-param capture + assertion (catches hardcoded-filter drift) — added during code review.
- `earnings-window.utils.spec.ts` (new): calendar-month boundaries, month/year rollover, leap February, from/to precedence.

**Web**
- `admin-earnings.spec.ts` (new):
  - Initial load via `AdminEarningsService` endpoint, default `1m` window.
  - "Last month" tab label includes the actual month name.
  - Window-tab selection issues the right query params.
  - Multi-currency rendering (ZAR + NAD both shown, never summed).
  - Empty states (zero vendors, zero-activity vendor row) and the load-error path.

**Counts:**
- `npm run test:api`: 571/571 pass (41 suites).
- `npm run test -w @hb/web`: 712/712 pass (58 files).
- `npm run lint`: both clean.
- `npm run build`: clean (shared → api → web).

## Key decisions worth recording

### 1. Per-vendor table scope (eligible-lines-only)

**What:** `VendorEarningsSummaryDto` rows sum only `accrued` + `settlementPreview` buckets, excluding `pendingClaimWindow` entirely.

**Why:** Card AC left scope open; explicit `AskUserQuestion` was filed before implementation. This decision aligns the vault's "commission on eligible lines" rule and ensures per-vendor totals reconcile with headline figures when all vendors are currently `APPROVED`.

**How:** `getEarningsByVendor` merges each vendor's `accrued` + `settlementPreview` buckets (both past the 48h claim-window boundary) into per-currency commission/net totals; `pendingClaimWindow` (still inside the 48h window) is never touched.

**Impact:** Per-vendor numbers are actionable for settlement validation; no silent surprises if admin is reconciling vendor payouts.

### 2. `platformCommissionByCurrency` includes inactive vendors

**What:** Headline commission sum includes any vendor with eligible activity in range, even if currently suspended/rejected.

**Why:** That commission is real historical H&B revenue regardless of vendor status.

**How:** Query joins `order_items` on `vendorId` without filtering by vendor approval status.

**Impact:** Headline sum can exceed Σ of per-vendor Commission column (intentional, documented on contract + UI caption). Tested explicitly.

### 3. `heldForVendorsByCurrency` is a snapshot, not temporal

**What:** Headline "held for vendors" is VE-3's `accrued` bucket as-of-now, NOT window-scoped. Querying past month still reports current holdings.

**Why:** This is the actual money locked in limbo at request time, independent of date range.

**Impact:** Caught in code review as a likely confusion point; documented on contract after the fact.

### 4. `vendorId` filter narrows the entire report

**What:** Optional query param `vendorId` filters all headline figures + per-vendor table.

**Why:** Least-surprising behavior for "show earnings for vendor X only".

**How:** Passed to both `AdminEarningsService` headline calculation and `VendorEarningsService.getEarningsByVendor()` call.

**Impact:** `platformListingGmvByCurrency` correctly resolves to empty for vendor filters (platform lines have no vendor owner, so no branch needed).

### 5. Code review caught two real bugs before merge

**a) Query-builder mock was hardcoding filters instead of capturing them**
- **Issue:** `getMany()` mock asserted `listingType = PLATFORM` and `status != CANCELLED` as inline, not from captured bind params. A dropped production filter would still pass tests.
- **Fix:** Now genuinely captures all bind params and asserts on each one. New test: vendor-line exclusion and non-delivered-order inclusion.

**b) SSR/hydration mismatch on "Last month" tab label**
- **Issue:** Label computed from `new Date()` at field-init without timezone. Server (UTC) and browser (local) can disagree on calendar month at boundaries.
- **Fix:** Label now derived from `report.from` field with explicit `timeZone: 'UTC'`.

### 6. `EARNINGS_WINDOWS` const sourced, not duplicated

**What:** Single `EARNINGS_WINDOWS` const array, validator uses `@IsIn(EARNINGS_WINDOWS)` instead of repeating the union literal.

**Why:** Code review flagged validator/consumer drift risk (one change breaks the other silently).

**Impact:** Future window additions or removals touch one place, propagate automatically to validator.

## Review & test outcome

**Code review pass** found 2 blocking issues, both fixed before merge:

1. **Query-builder filter capture** — mocked filters hardcoded inline. Fixed: capture all bind params, assert on each; new test verifies filters apply.
2. **SSR tab-label hydration mismatch** — label from client clock. Fixed: label from `report.from` with explicit `timeZone: 'UTC'`.

**Non-blocking advisory applied:**
- Contract doc-comments on `platformCommissionByCurrency` and `heldForVendorsByCurrency` clarified to surface the unintuitive behaviors (includes inactive vendors, held is snapshot not temporal).

**Test results:**
- `npm run test:api`: 571/571 pass (41 suites).
- `npm run test -w @hb/web`: 712/712 pass (58 files).
- `npm run lint:api` / `npm run lint -w @hb/web`: both clean.
- `npm run build`: clean (shared → api → web).

## What's not in scope

- CSV export or scheduled report delivery (separate future cards).
- Payout execution or settlement job (entirely out of scope, separate TBD).
- VE-5 (vendor own-earnings — parallel sibling card, depends on VE-3 like this one, not on this card).

## Follow-ups

### Next in chain:

- **VE-5** (#74, `WrPOgQDa`) — Vendor own-earnings: `GET /vendors/me/earnings` endpoint + vendor portal screen. Depends on VE-3, same as this card — NOT blocked by VE-4. Can opportunistically reuse `resolveEarningsWindow` and the `getEarningsByVendor`/eligible-bucket-merging pattern this card introduced.
- No UI `vendorId` filter control was built in this slice — the API supports it; add a filter input if an admin workflow needs it.

### No open items from this card:

- `EARNINGS_WINDOWS` is single-sourced, validator drift prevented.
- Per-vendor scope is documented and tested (eligible-lines-only).
- Inactive-vendor inclusion is tested and explained.
- SSR hydration fixed on "Last month" label.
- Query-builder test coverage is genuine (bind params captured, not hardcoded).

## PR link

Trello card 73: https://trello.com/c/1DAb9a1I  
Branch: `feat/1DAb9a1I-admin-earnings-report`  
PR: https://github.com/michaeljvr11/hb-mono-repo/pull/47
