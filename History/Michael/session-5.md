---
operator: Michael
date: 2026-06-22
session: 5
tags: [ai-factory, ship-card, admin-portal]
---

# Session 5 — Admin audit log (activity trail)

## What we worked on
Trello card **AP-11 / 7oPc1cnk** — "Admin portal — audit log (activity trail + admin UI)". Shipped activity logging across API and web in one PR.

## What changed

**Backend:**
- `@hb/shared` contracts added: `AuditLogDto`, `AuditLogListQuery`, `AuditLogListDto`.
- `apps/api/src/audit/`: new `@Global() AuditModule` with `AuditService.log()` (best-effort, try/catch, never throws) and `query()` (server-side filtering + pagination).
- `AuditLog` entity: `audit_logs` table with `id` (uuid), `userId` (FK), `action` (varchar), `entityType` (varchar), `entityId` (varchar), `metadata` (jsonb), `createdAt` (timestamptz); indexed on action/entityType/userId/createdAt.
- Migration `1782086400000-AuditLog.ts`: creates table with `gen_random_uuid()`, symmetric up/down.
- `AuditService.log()` wired into: `AdminService.updateUserRole` (`'user.role_assigned'`), `AdminService.setUserActive` (`'user.activated'`/`'user.deactivated'`), `VendorsService.updateStatus` (`'vendor.status_changed'`), `ProductsService` create/edit/delete (`'product.created'`/`'updated'`/`'deleted'`), `AuthService.bootstrapAdmin` (`'admin.bootstrapped'`).
- `GET /admin/audit-logs`: admin-gated, filterable by action/entityType/userId/from/to, paginated. Server-side filtering; date-only `to` normalized to end-of-day.
- 12 new Jest unit tests covering best-effort log, query filters, pagination, metadata preservation; wiring assertions for vendor status, user role/activation, admin bootstrap.
- Schema change: the new `AuditLog` entity ships with its TypeORM migration (`synchronize` stays off).

**Frontend:**
- New `AuditService` in `core/api/` wraps `GET /admin/audit-logs`.
- Replaced `admin-logs` placeholder with real signals-based screen: filterable activity table (timestamp / actor userId / action / entityType / entityId columns), filter form (action dropdown, entityType dropdown, date-range picker, userId input), pagination (Prev/Next), loading/empty/error states.
- Standalone, signals-based, SSR-safe throughout; styled to `docs/design/DESIGN.md` tokens; mirrors `admin-users` / `admin-vendors` pattern.
- 24 Vitest specs covering filters, pagination, table rendering, empty/error/loading states.
- No `@hb/shared` contract changes beyond audit DTOs.

## Key decisions

**`@Global() AuditModule`**: audit is cross-cutting (vendors, products, users, orders all emit events), so a `@Global()` module avoids circular imports. Proven pattern in the codebase (`TypeOrmModule` is `@Global()`).

**Best-effort `log()`**: try/catch + `Logger.warn`, never throws into the caller. Audit writes must never break the business action that triggered them. If the audit table is full or the connection drops, the business-critical action succeeds and ops gets a warning.

**Order-status logging deferred**: the orders module is still a skeleton with no real mutations. Wire order transitions when VP-4 (checkout) or order-mutation cards land.

**Actor by `userId` only**: no email enrichment yet. Admin sees "User ID `abc-123` activated user `def-456`" — correct and unblocking. Enrichment is a follow-up.

**Date-only `to` normalized to end-of-day**: if the admin selects 2026-06-22 as the upper bound, they expect to see all actions on that day. The query treats `to: "2026-06-22"` as `createdAt <= 2026-06-22 23:59:59.999Z`.

**Metadata untyped in the response**: `AuditLogDto.metadata?: Record<string, unknown> | null`. Specific shapes (`{ role }` for role changes, `{ from, to }` for vendor status; user-active/product/bootstrap log no metadata) live in service code, not enforced at the contract boundary. A metadata diff renderer is a follow-up.

## Test/review outcome
- **api 106/106** — all tests pass (12 new for audit service + wiring assertions in admin/vendor tests).
- **web 272/272** — all tests pass (24 new for admin-logs screen).
- **Lint clean**, **full SSR build pass** (only pre-existing SCSS budget warnings on `admin-catalog/shop.scss`).
- **Code review:** SHIP — one date-filter WARN fixed (date-only `to` end-of-day) + nits; awaiting human merge via PR #17.

## Follow-ups
- Wire order-status logging once order mutations exist (VP-4 / checkout card).
- Actor-email enrichment — either endpoint-side join to users table or client-side mapping service.
- Metadata key/value diff renderer in the admin-logs UI (e.g., "Customer → Vendor" for role changes).
- Search/export audit trail (full-text search + CSV download, later UX niceties).
- `products.service.spec.ts` for fuller product-audit coverage (product create/edit/delete tests in isolation).

## PR status
PR #17 — in review. Awaiting human merge (lead owns prod).

## Orchestration
- **backend-engineer slice**: `@hb/shared` contracts, `AuditModule`, entity + migration, wiring + unit tests.
- **frontend-engineer slice**: `AuditService`, admin-logs screen, Vitest specs.
- **full test/lint/build gate**: 106 api + 272 web + lint clean + SSR clean.
- **code-reviewer slice**: SHIP.
- **docs slice** (this session): Obsidian implementation note + session log + evidence snapshot.
