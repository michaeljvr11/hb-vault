# Session 8 — 2026-06-20

## Card worked
VP-2 — Vendor portal dashboard screen + read-model endpoint (`feat/oWqIV57s-vendor-dashboard`, PR #15)

## What changed
Full-stack: `@hb/shared` `VendorDashboardDto`, API `GET /vendors/me/dashboard`, Angular `VendorDashboard` component replacing the placeholder.

Key changes:
- `libs/shared/src/contracts/vendor.ts` — `VendorDashboardDto` interface
- `apps/api/src/vendors/vendors.module.ts` — added `Product` + `OrderItem` to `TypeOrmModule.forFeature`
- `apps/api/src/vendors/vendors.service.ts` — `getDashboard(userId)` with revenue aggregation
- `apps/api/src/vendors/vendors.controller.ts` — `GET /vendors/me/dashboard`
- `apps/api/src/vendors/vendors.service.spec.ts` — 7 new revenue aggregation unit tests
- `apps/web/src/app/core/api/vendors.service.ts` — `getDashboard()` method
- `apps/web/src/app/features/vendor/pages/vendor-dashboard/` — full component (ts / html / scss / spec)

## Key decisions
- Revenue = SUM(unitPrice * quantity) across all order_items for this vendor. Null aggregate guarded twice.
- Order counts are per line-item, not per distinct order — matches the fulfilment-visibility use case; UI copy says "Order lines".
- Currency hardcoded ZAR; NAD mixing deferred to a future card.
- Direct entity repo injection in VendorsModule (same pattern as AdminModule in AP-9) — no circular deps.

## Test/review outcome
- api 72/72, web 175/175, lint clean, full SSR build clean
- Code review verdict: **SHIP** — no FAILs; two non-blocking WARNs (line vs order count semantics; no SQL expression assertion)

## Follow-ups
- Distinct-order count in dashboard once checkout path exists (VP-4 dependency)
- NAD mixed-currency aggregation
- `toSignal` / `takeUntilDestroyed` cosmetic refactor
