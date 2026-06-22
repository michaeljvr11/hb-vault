---
operator: Michael
date: 2026-06-22
session: 4
tags: [ai-factory, ship-card, admin-portal]
---

# Session 4 — Admin order oversight + dashboard metrics

## What we worked on
Trello card **AP-10 / sw8qYBV1** — "Admin portal — order oversight + dashboard metrics". Shipped read-only order visibility across API and web in one PR.

## What changed

**Backend:**
- `@hb/shared` contracts added: `AdminDashboardDto`, `CurrencyTotalDto`, `OrderStatusCountDto`, `AdminOrderListItemDto`, `AdminOrderListDto`, `AdminOrderListQuery`.
- `AdminModule` now registers `Order` / `OrderItem` / `Vendor` repos directly (no module imports) to avoid circular dependencies.
- `GET /admin/dashboard` — pending-vendor count, total orders, order counts zero-filled across every `OrderStatus`, and platform vs vendor revenue **grouped by currency** (ZAR and NAD never summed per [[Money & Currency Rules]]). Cancelled orders excluded. Numeric strings coerced via `Number()`.
- `GET /admin/orders` — paginated list across all users/vendors, filterable by `status` and `vendorId`. Vendor-filtered orders keep full item sets so `itemCount` and `vendorIds` stay accurate. In-memory pagination at skeleton stage; DB-level pagination deferred.
- 24 new Jest unit tests: revenue-aggregation suite (platform/vendor split, per-currency grouping, cancelled excluded, numeric coercion, zero-data empties).
- No schema change, no migration.

**Frontend:**
- New `AdminOrdersService` in `core/api/` with `getDashboard()` and `listOrders(query)` (HttpParams built only from defined fields).
- Replaced Dashboard placeholder with real signals-based screen: metric cards (pending-vendors CTA, total orders, orders-by-status, per-currency platform/vendor revenue). Empty state renders zeros / "No orders yet" — never an error.
- Replaced Orders placeholder with paginated table (server-side `status` + `vendorId` filters, Prev/Next pagination, inline split list/detail panel). Zero-data state friendly, expected while orders module is skeleton.
- Standalone, signals-based, SSR-safe throughout; styled to `docs/design/DESIGN.md` tokens; mirrors `admin-users` / `admin-vendors` pattern.
- +66 Vitest specs.
- No `@hb/shared` contract changes — reused existing `@hb/shared` enums and added new DTOs as above.

## Key decisions

**Read-only scope:** AP-10 exposes no admin order *actions* (no cancel, no override). Admin mutations will land in a follow-up once the orders/checkout module is real. Audit log (AP-11) is listed as "best shipped after" — not a blocker since there are no actions to log yet.

**Revenue model:** Line-item GMV (`unitPrice * quantity`), grouped by currency, excluding cancelled orders. Shipping fees excluded (documented on `CurrencyTotalDto`). ZAR and NAD kept as separate entries, never summed — the 1:1 peg is data, not an assumption (per [[Money & Currency Rules]]).

**Pagination:** In-memory at skeleton stage; DB-level pagination + query-level `vendorId` filter deferred to when order volume lands and the orders module becomes real.

**Module architecture:** `AdminModule` registers repos directly (same pattern as VP-2 vendor dashboard) to avoid circular imports; `UsersModule` / `VendorsModule` / `ProductsModule` are not imported.

## Test/review outcome
- **api 86/86** — all tests pass (24 new).
- **web 207/207** — all tests pass (+66 new).
- **Lint clean**, **full SSR build pass** (only two pre-existing SCSS budget warnings, unrelated).
- **Code review:** SHIP WITH NITS. Nits addressed in-PR: currency-symbol `Record` lookup with safe fallback, `CurrencyTotalDto` shipping-exclusion doc, test typo.

## Follow-ups
- DB-level pagination + query-level `vendorId` filter when checkout/order volume lands.
- Dedicated `/admin/orders/:id` order-detail route once the orders/checkout module is real.
- "Paid-only" revenue refinement (filter by payment status) once payments are real.
- **This completes slice 10 of the admin-portal sequence.**

## PR status
PR #16 — in review. Awaiting human merge (lead owns prod).
