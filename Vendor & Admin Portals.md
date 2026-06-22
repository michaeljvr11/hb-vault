# Vendor & Admin Portals

Spec for the two role-gated back-office surfaces on top of the existing
role-discriminated identity model. Front-of-funnel planning only — implementation
flows through `/ship-card`. Related: [[HB Domain Model]] · [[Auth & Roles]] ·
[[Listing Types & Vendor Rules]] · [[Order State Machine]] · [[Money & Currency Rules]].

## Problem

Customers can browse/shop today, but there is **no surface for vendors to run their
business** (list products, see their orders) and **no surface for admins to run the
platform** (approve vendors, oversee listings/users/orders). The data model and most
vendor/admin API endpoints already exist; what's missing is the portal UI and a few
backend gaps (vendor order/fulfilment, admin oversight, dashboard read-models).

## Architecture decisions (confirmed with the user, 2026-06-18)

1. **Identity stays on the single role-discriminated `users` table** (`customer | vendor
   | admin`). No separate admin store. A vendor's business profile remains the existing
   `vendors` row, `OneToOne` to the user via `userId`. See [[Auth & Roles]].
2. **One NestJS monolith.** Reuse the settled `@Roles` / global `RolesGuard` + the
   service-layer ownership pattern. No microservices, no new auth flow.
3. **Frontend = lazy-loaded, role-gated route areas inside `apps/web`** — `/vendor/**`
   (`roleGuard(['vendor'])`) and `/admin/**` (`roleGuard(['admin'])`), each with its own
   shell/layout component. **Not** separate apps or subdomains. Subdomains remain a
   later deployment option requiring zero data-model change (same registrable domain keeps
   the refresh cookie working — see [[Auth & Roles]] SameSite note).
4. **Sequencing: vendor portal first.** Admin **vendor-approvals** is its gating
   dependency (a vendor can't transact until an admin approves them).

## Business rules it must honour

From [[Listing Types & Vendor Rules]] and [[Auth & Roles]] — enforce, don't reinvent:

- **Only `approved` vendors can list products or receive orders.** Lifecycle:
  `pending → approved` (↘ `rejected`); `approved → suspended` (and back). Status changes
  are **admin-only**.
- **Vendor-facing queries MUST filter by ownership** — a vendor sees only their own
  listings/orders. Ownership checks live in the **service layer**, never hand-rolled in
  controllers.
- **Listing-type invariant:** vendors create `vendor` listings (must have a `vendorId`);
  admins create `platform` listings (must not). DB CHECK already enforces this; don't
  bypass it.
- **Admin scope:** manage categories, platform listings, vendor approval/suspension, and
  **all** orders. Vendor scope: own profile, own listings, own order lines.
- Money stays `numeric(12,2)` + explicit currency; ZAR/NAD peg is data. Dashboard
  aggregates that sum money get a unit test. See [[Money & Currency Rules]].

## What already exists (do NOT re-build)

- **Vendor API:** full CRUD, `POST /vendors` (self-onboard), `POST /vendors/admin`,
  `PATCH /vendors/:id/status` (approve/reject/suspend), `GET /vendors` (admin list),
  `GET /vendors/me`. (`apps/api/src/vendors/vendors.controller.ts`)
- **Products API:** vendor/admin create with ownership-scoped edit/delete; public read.
- **Categories API:** admin CRUD.
- **Auth/guards:** `JwtAuthGuard` + `RolesGuard` + `@Roles`/`@Public`; web `authGuard` +
  `roleGuard` (reads `route.data.roles`). Vendor route already scaffolded (commented) in
  `apps/web/src/app/app.routes.ts`.
- **Design:** a `vendor-dashboard` screen is already pulled under `docs/design/`.

## Backend gaps to fill

- **Vendor orders / fulfilment:** `orders` currently exposes only customer `GET /orders`
  (`getMyOrders`). Need vendor-scoped read of the `order_items` belonging to the vendor +
  the confirmed fulfilment transitions (`confirmed→processing`, `processing→handed_to_hb`).
  See [[Order State Machine]] for the full confirmed transition table.
- **New `handed_to_hb` OrderStatus:** a new enum value in `@hb/shared` + migration is
  required (see [[Order State Machine]]).
- **Admin module:** no consolidated `admin/` module. Admin user-management (list/search,
  activate/deactivate, role assignment) and cross-cutting order oversight have no home.
- **Dashboard read-models:** aggregate endpoints for vendor dashboard (own counts, recent
  orders, revenue) and admin dashboard (pending-vendor count, totals).
- **Admin bootstrap endpoint:** `POST /auth/bootstrap-admin` — self-sealing (refuses if
  any admin already exists). Accepts `email` + `password`, creates the first admin user.
  Callable from Bruno. Must have a unit test verifying it refuses on the second call.

- **Audit log:** new `audit_logs` entity (`id`, `userId`, `action`, `entityType`, `entityId`, `metadata` JSONB, `createdAt`) + TypeORM migration. A shared `AuditService.log()` helper called from service methods on key actions: vendor status changes, order transitions, user role/activation changes, product create/edit/delete, admin bootstrap. Admin-gated `GET /admin/audit-logs` with filtering by action, entityType, userId, and date range + pagination.

## `@hb/shared` contract impact

Interfaces + enums only; API DTOs `implement` them; web consumes them. Likely additions
(confirm exact shape per slice):

- `enums/order-status.ts`: add `handed_to_hb` value.
- `contracts/order.ts`: vendor-scoped order/line view DTO; vendor fulfilment status-update
  request DTO (only `confirmed→processing` and `processing→handed_to_hb` allowed).
- `contracts/user.ts`: admin user-list item + update-role / set-active request DTO.
- New `contracts/dashboard.ts`: vendor + admin dashboard summary DTOs.
- Vendor contracts already cover onboarding/status (`CreateVendorRequest`,
  `AdminCreateVendorRequest`, `UpdateVendorStatusRequest`). Reuse — don't duplicate.

Every new endpoint input = a class-validator DTO implementing the shared interface.

- `contracts/audit.ts` (new): `AuditLogDto` (`id`, `userId`, `action`, `entityType`, `entityId`, `metadata`, `createdAt`); `AuditLogQueryDto` for filter params (`action?`, `entityType?`, `userId?`, `from?`, `to?`, `page`, `limit`).

## Resolved decisions (2026-06-18)

| Question | Decision |
|---|---|
| Who can trigger which order transition? | Admin: all. Vendor (own lines only): `confirmed→processing`, `processing→handed_to_hb`. HB/admin then handles `handed_to_hb→shipped→delivered`. |
| Vendor onboarding depth for v1? | No document upload. Existing `CreateVendorRequest` fields are sufficient. `verificationDocumentUrl` stays optional, KYC deferred. |
| Admin bootstrap? | One-shot `POST /auth/bootstrap-admin` endpoint — refuses if any admin exists (self-sealing), returns a normal auth response. Callable from Bruno. |

| Audit log type? | Activity trail stored in a new `audit_logs` DB table. Capture: vendor status changes, order transitions, user role/activation changes, product edits, admin bootstrap. Not application/error logs. |

## Out of scope (separate cards / TBD)

- KYC / banking / verification-document review — deferred past v1 per [[Listing Types & Vendor Rules]].
- Commission & fee structure, vendor cross-border self-fulfilment — TBD.
- Splitting admin into its own `apps/admin` / subdomain — deferred deployment option.
- Real payment/courier providers (ports + stubs stay stubs without an explicit card).
- Order-placement gate on `isVerified` — already its own (blocked) card #15.
- Cancellation after `processing`, partial shipment/delivery — still TBD in [[Order State Machine]].

## Open questions (remaining)

- Cancellation policy after `processing` — who can cancel, any restocking fee?
- Mixed-line orders where one vendor ships and another hasn't — how does order status aggregate?

## Vertical slices (→ Trello cards)

**Vendor portal (first)**
1. Vendor route area + shell/layout + `roleGuard(['vendor'])`.
2. Vendor dashboard screen wired to a vendor-summary read-model endpoint.
3. Vendor product management UI.
4. Vendor orders & fulfilment — `GET /orders/vendor` + `confirmed→processing` + `processing→handed_to_hb` transitions.
5. Vendor onboarding UI — "become a vendor" → `POST /vendors` (no doc upload in v1).

**Admin portal**
6. Admin route area + shell/layout + `roleGuard(['admin'])` + **admin bootstrap endpoint**.
7. Vendor approvals (keystone) — gating dependency for vendor portal.
8. Admin catalog (platform listings) + category management UI.
9. Admin user management — `admin/` module: list/search, activate/deactivate, role assign.
10. Admin order oversight + dashboard metrics.
11. Admin audit log — `audit_logs` table + `AuditService` + admin UI to view/filter the activity trail.

## Implementation notes

### Admin portal foundation — shell + bootstrap endpoint — 2026-06-18 (card AP-6 / Z1ADCLQm)

**Backend — `POST /auth/bootstrap-admin`** (`@Public`): self-sealing one-time admin seed. Counts admins via new `UsersService.countByRole`; returns **409 Conflict** once any admin exists. Cannot be re-exploited. On success creates an admin user with `isVerified: true` (trusted setup action, exempt from order-gating verification) and returns the normal auth response — access token + httpOnly refresh cookie, identical to `/auth/login`. New shared contract `BootstrapAdminRequest`; `BootstrapAdminDto implements` it with class-validator `IsEmail` and `MinLength(8)`. Password hashed by the `User` `@BeforeInsert` hook like register. Bruno request "Bootstrap Admin" added for one-click first-time setup. Unit test asserts the 409-on-second-call gate (non-negotiable), admin creation, and email-collision (400).

**Frontend — `/admin` route area**: lazy, `canActivate: [authGuard, roleGuard]`, `data: { roles: ['admin'] }` — unauthenticated and non-admin users are redirected to `/login` by the existing guards. Standalone signals-based, SSR-safe `AdminShell` (sticky top bar with H&B Market wordmark + "Admin Console" badge + sign-out; sidebar nav: Dashboard / Vendors / Catalog / Users / Orders / Logs with active-link highlight; `<router-outlet>` for children). Six lazy placeholder child pages; default child redirects to `dashboard`. Styled to the `admin-dashboard` Claude Design tokens; three new tokens registered in `docs/design/DESIGN.md` (`--hb-secondary-fixed`, `--hb-on-secondary-container`, `--hb-scrim`).

**No schema change** — `users.role` and `users.isVerified` already exist.

**Tests/review:** api 25/25, web 46/46, lint clean, full build clean. Code review: SHIP, no FAIL items (design-token nits addressed).

**Follow-ups:** real admin screens replace the placeholders in later AP cards (AP-7 vendor approvals is next — the keystone).

### Vendor portal foundation — route area + shell — 2026-06-19 (card VP-1 / EXPtZCJL)

**Frontend only** — no backend changes. Uncommented and completed the `/vendor` lazy route in `apps/web/src/app/app.routes.ts`: `canActivate: [authGuard, roleGuard]`, `data: { roles: ['vendor'] }`, `loadComponent: () => VendorShell`, with four lazy child routes (`''` redirects to `dashboard`, plus `dashboard`, `products`, `orders`, `profile`).

New standalone signals-based `VendorShell` component at `apps/web/src/app/features/vendor/vendor-shell/` — mirrors the merged `AdminShell` pattern exactly: sticky top bar with H&B Market wordmark + "Vendor Portal" badge + sign-out button; sidebar nav (Dashboard / Products / Orders / Profile) with active-link highlight; `sidebarOpen` signal + toggle/close on nav; `<router-outlet>` for child routes. SSR-safe — no browser APIs, only signals.

Four lazy placeholder pages under `apps/web/src/app/features/vendor/pages/` (dashboard, products, orders, profile) — each returns "Coming soon" so child routes resolve without errors and later screens can swap in without route changes.

Vitest spec mirrors `admin-shell.spec.ts`: renders shell, validates four nav labels + paths, verifies sidebar toggle/close, confirms signOut calls `AuthService.logout()`.

Non-vendor roles (`customer`, `admin`) and unauthenticated users are already redirected to `/login` by the existing `authGuard` + `roleGuard` — no new guard code needed.

**Tests/build:** `npm run test -w @hb/web` 72/72 pass; `npm run lint:api` clean; `npm run build` (SSR) full pass. Code review verdict: **SHIP**, no FAIL items.

**Out of scope (deferred):** real vendor dashboard/products/orders/profile screens, vendor-summary read-model endpoint, design polish, vendor onboarding flow (VP-5).

### Vendor approvals (keystone) — 2026-06-19 (card AP-7 / DD6Z0NUW)

**Backend (`@hb/shared` + API).** The admin approvals UI needs onboarding fields the public `VendorResponseDto` deliberately hides. Added `AdminVendorDto extends VendorDto` to `@hb/shared` (adds `registrationNumber?`, `website?`, `description?`, `verificationDocumentUrl?`, `appliedAt`). New `AdminVendorResponseDto implements AdminVendorDto` on the API. `VendorsService.findAll()` (the admin-only `GET /vendors`) now returns the richer shape via a new `toAdminResponseDto` mapper; `appliedAt` is `createdAt.toISOString()`. The public `GET /vendors/directory` and `GET /vendors/:id` stay on the lean `VendorResponseDto`, so registration numbers / verification docs never leak to non-admins. **No schema change** — every field already exists on the `vendors` entity.

**Frontend.** Replaced the `admin/vendors` placeholder with the real keystone screen (`apps/web/src/app/features/admin/pages/admin-vendors/`): status filter tabs (All / Pending / Approved / Rejected / Suspended), a vendor list, and a detail panel (business/trading name, registration number, country, applied date, and `verificationDocumentUrl` as a link only when present). Status lifecycle is enforced in the UI by a pure, unit-tested `vendorActionsFor(status)` helper so only valid transitions are offered (pending→approve/reject, approved→suspend, suspended→re-approve, rejected→none); actions call `PATCH /vendors/:id/status` and update local signal state in place. Standalone, signals, new control flow, SSR-safe, styled to DESIGN.md tokens. `VendorsService.list()` retyped to `AdminVendorDto[]`.

**Tests/review:** api 30/30, web 86/86, lint clean, full SSR build pass. Code review verdict **SHIP** — two WARNs fixed in-PR (honoured the `appliedAt: string` contract by dropping a dead null-guard; swapped a hardcoded error-banner hex for `--hb-error` + `color-mix`) plus a double-submit guard test.

**Follow-ups:** unblocks real end-to-end vendor-portal testing. Document upload / KYC review still deferred (future card). Next admin card: AP-8 admin catalog.


### Vendor approvals — server-side lifecycle enforcement — 2026-06-19 (card RLiauFte, PR #9)

AP-7 (above) shipped the approvals UI, but the status lifecycle lived **only** in the Angular
`vendorActionsFor` helper. `VendorsService.updateStatus` persisted whatever status it was
handed, so a direct `PATCH /vendors/:id/status` bypassed the keystone gate entirely — e.g.
resurrecting a `rejected` vendor or re-approving an already-`approved` one. Since approval is
the gating dependency for the whole vendor portal, that is a real authorization hole.

Closed it by enforcing the lifecycle in the **service layer** (where the spec requires it):
a `STATUS_TRANSITIONS` table mirroring the client one — `pending→approved/rejected`,
`approved→suspended`, `suspended→approved`, `rejected` terminal — with illegal transitions
rejected as **409 Conflict** before any write. The controller already carries
`@Roles(UserRole.ADMIN)`, so the guard is admin-only. **No schema or contract change** (reuses
`UpdateVendorStatusRequest`); the existing DTO whitelist and this from-state guard are
complementary. 11 new unit tests cover the valid matrix, every illegal transition, and
not-found. api 42/42, lint + build clean; code review **SHIP**.

This satisfies the last unchecked requirement of slice 7 ("admin-only, service-layer
validation"). Slice 7 is now complete end-to-end (UI = PR #8, server guard = PR #9).

### Admin catalog — platform listings + category management UI — 2026-06-19 (card AP-8 / IhTVdNYP)

**Frontend only** — no backend or `@hb/shared` changes; the product/category CRUD endpoints and the web `ProductsService`/`CategoriesService` already existed and were reused.

Replaced the `admin/catalog` placeholder with a real screen at `apps/web/src/app/features/admin/pages/admin-catalog/` (`.ts`/`.html`/`.scss`/`.spec.ts`), mirroring the merged `admin-vendors` pattern: standalone, signals-first, `inject()`, DESIGN.md tokens, separate template/style files, loading/error/empty states.

Two sections behind a tab switcher: **Platform Listings** and **Categories**.

**Listings:** list is a `computed()` filtered to `listingType === ListingType.PLATFORM` (vendor listings never shown). Create → `POST /products` (multipart image upload via existing disk-storage flow), edit → `PATCH /products/:id`, delete → `DELETE` with an inline two-step confirm. `vendorId` is never exposed or sent — the create/update payloads are explicit object literals that omit it; the server forces `listingType: 'platform'` for admins. (Note: the PATCH path does not handle image changes — only create uploads images.)

**Categories:** full CRUD via the admin category endpoints (`name`, `slug?`, `description?`, `displayOrder?`, `parentId?`).

Tests/review: web 119/119 pass, full SSR build clean (one non-blocking SCSS budget warning, same class as the tolerated `shop.scss`). Code review: **SHIP**, no FAILs; addressed a review nit by switching two hardcoded shadow `rgba()` literals to `color-mix` over tokens.

**Follow-ups:** image editing on PATCH and an SCSS budget trim are deferred polish. Next admin cards: AP-9 (user management + `admin/` module), AP-10 (order oversight), AP-11 (audit log).


### Admin portal — user management + admin NestJS module — 2026-06-19 (card AP-9 / N8P6OPPm)

**Branch:** `feat/N8P6OPPm-admin-user-mgmt`
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/11
**Status:** In Review

#### What shipped
- **`@hb/shared`**: Four new interfaces — `AdminUserDto` (extends `UserDto` + `isActive`, `createdAt`), `UpdateUserRoleRequest`, `SetUserActiveRequest`, `AdminUserListQuery`
- **`apps/api/src/admin/`**: New `AdminModule` with `AdminService` + `AdminController` protected by `@Roles(UserRole.ADMIN)`. Endpoints: `GET /admin/users` (search + filter by role/isActive), `PATCH /admin/users/:id/role`, `PATCH /admin/users/:id/active`. Self-demotion guards: admins cannot change their own role or deactivate their own account.
- **`apps/web`**: `UsersService` (HttpClient wrapper) + `AdminUsersComponent` with five filter tabs (All/Customer/Vendor/Admin/Inactive), text search, split list/detail layout, inline role `<select>`, Activate/Deactivate toggle with double-submit guard via `pendingId` signal.

#### Tests
- 17 new Jest unit tests in `admin.service.spec.ts` — list/filter/search, role update, activate/deactivate, self-demotion guards, not-found paths. 62 total passing.
- 20 Vitest specs in `admin-users.spec.ts`.

#### Key decisions
- `AdminModule` uses `TypeOrmModule.forFeature([User])` and injects `Repository<User>` directly — avoids circular import that would result from importing `UsersModule`.
- `requestingUserId` passed as explicit parameter to service methods (not extracted inside service) — keeps service testable without JWT context.
- Self-deactivation blocked; self-role-change blocked; re-activation of own account allowed.

### Vendor onboarding — "become a vendor" flow — 2026-06-19 (card VP-5 / 1fqtc2JF)

**Scope shift (important).** The card was scoped "frontend only," but the backend did **not** support the described flow. Confirmed full-stack with the product owner; onboarding was hardened across API + web in one PR. Divergences fixed: `POST /vendors` auto-approved (status APPROVED, not pending), was gated to `@Roles(vendor)` so a customer couldn't apply, returned 403 (not 409) on duplicate, and nothing ever granted the `vendor` role (so even approved vendors couldn't reach the role-gated `/vendor` portal).

**Backend (`apps/api/src/vendors`).** `POST /vendors` now `@Roles(CUSTOMER, VENDOR)` so an authenticated customer can self-onboard. `VendorsService.create()` sets `status: PENDING`, grants the applicant `role = vendor` via `UsersService.update` (idempotent — only when not already a vendor) so they enter the portal and can read `GET /vendors/me`, and throws **409 Conflict** (was 403) on a duplicate application. `VendorsModule` now imports `UsersModule` (verified acyclic — `UsersModule` imports only `TypeOrmModule.forFeature([User])`, the same leaf edge `AuthModule` already uses; no circular dependency, unlike the AP-9 AdminModule case). **No schema change / no migration** — the `vendors.status` column already defaults to `pending` and `users.role` already exists.

**Why no token rotation is needed.** `JwtStrategy.validate` re-loads the user from the DB on every request, so the DB role flip takes effect immediately for API authorization; the web client only needs to re-fetch `/users/me` to update its cached role for the route guard.

**Frontend (`apps/web`).** New `/vendor/apply` route guarded by `authGuard` only and **placed before** the role-gated `/vendor` parent (so a customer reaches it instead of being bounced by `roleGuard`). New standalone, signals-based `VendorOnboarding` component: customers get a typed reactive application form (`businessName` required; `tradingName` / `registrationNumber` / `countryCode` optional; **no document upload**) → `POST` `CreateVendorRequest` → refresh cached user (now vendor) → pending confirmation; a 409 surfaces a friendly "already applied". Vendors get a status screen from `GET /vendors/me` with per-status messaging (pending / approved / rejected / suspended; approved links through to `/vendor`). Added `AuthService.refreshCurrentUser()` and a "Sell on H&B" entry link in the shop nav bar. **No `@hb/shared` contract changes** — reused `CreateVendorRequest` / `VendorDto` / `VendorStatus` / `CountryCode`.

**Product decision.** A customer self-applying immediately receives `role = vendor` with `status = pending`. Pending gates nothing real — the public directory lists only `approved` vendors, status transitions are admin-only, and vendor-profile update stays owner-only — so this is safe.

**Tests/review.** api 45/45, web 141/141, lint clean, full SSR build clean (only the two pre-existing SCSS budget warnings, unrelated). Code review verdict **SHIP WITH NITS**, nits addressed in-PR (typed the test DTO to clear `no-unsafe-argument` warnings; made the 409 path load vendor status even if the user refresh errors).

**Follow-ups.** Status-gate the vendor portal feature pages (dashboard/products/orders placeholders) by vendor status so a pending/rejected vendor can't use approved-only features; `vendors.businessName` is a `unique` column, so a duplicate business name currently yields a raw 500 rather than a friendly message (future card); document upload / KYC still deferred. This completes **slice 5** of the vendor-portal sequence.

### Vendor product management UI — 2026-06-19 (card VP-3 / xEOk5i0M)

**Branch:** `feat/xEOk5i0M-vendor-products`
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/14
**Status:** In Review

#### What shipped
**Frontend only** — no backend or `@hb/shared` changes.

Replaced the `VendorProducts` placeholder at `apps/web/src/app/features/vendor/pages/vendor-products/` with a full CRUD screen. The component loads `GET /vendors/me` to resolve the authenticated vendor's ID, then filters `GET /products` client-side to show only that vendor's own listings (`product.vendor?.id === vendorId()`). This keeps the trust boundary on the server (ownership enforced by `ProductsService`) while providing an appropriate vendor-scoped UX.

**Create flow:** form calls `POST /products` with no `vendorId` in the payload — the server's `createWithImages` resolves `vendorId` from the auth token and forces `listingType: 'vendor'`. Image upload via `multipart/form-data` (existing `FilesInterceptor` + `uploads/` disk-storage flow). **Edit flow:** `PATCH /products/:id` (no image changes, matching admin-catalog). **Delete:** `DELETE /products/:id` with 2-step inline confirm; failed deletes surface an error banner (code-review WARN addressed).

Standalone, signals-based, SSR-safe component; styled to `docs/design/DESIGN.md` tokens; follows the merged `AdminCatalog` pattern exactly (drawer form, table, SCSS conventions). Categories loaded for the form from `GET /categories`.

**Tests/review:** 183/183 Vitest web tests pass; `npm run lint:api` clean; full SSR build clean. 21 new specs covering vendor-only filtering, create (asserts `vendorId` absent from payload — server must set it), edit, delete success + error paths. Code review verdict: **SHIP**, no FAIL items; two WARNs addressed in-PR (delete-error banner + delete-error tests).

**Follow-ups:** image editing on PATCH (deferred, matching admin-catalog); category management is admin-only and not replicated here.

### Vendor portal dashboard screen + read-model endpoint — 2026-06-20 (card VP-2 / oWqIV57s)

**Branch:** `feat/oWqIV57s-vendor-dashboard`
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/15
**Status:** In Review

#### What shipped
**Full-stack** — `@hb/shared` contract + API endpoint + Angular UI.

**`@hb/shared`**: new `VendorDashboardDto` interface (`productCount: number`, `orderCountByStatus: Partial<Record<OrderStatus, number>>`, `totalRevenue: number`, `currency: CurrencyCode`) added to `libs/shared/src/contracts/vendor.ts`.

**API (`apps/api/src/vendors`)**. `VendorDashboardResponseDto implements VendorDashboardDto`. `Product` and `OrderItem` repos injected directly into `VendorsService` via `TypeOrmModule.forFeature([Vendor, Product, OrderItem])` — same pattern as `AdminModule` (no circular deps). `getDashboard(userId)` sequentially: (1) resolves the vendor row by userId (throws 404 if none); (2) counts products with `productRepository.count({ where: { vendorId } })`; (3) aggregates total revenue with `SUM(CAST(oi.unitPrice AS decimal) * oi.quantity)` via QueryBuilder on `order_items` filtered by `vendorId`; (4) groups order-line counts by order status. Null aggregate → 0 (double-guarded: `?? '0'` + `|| 0`). Currency hardcoded `ZAR` (all order lines default to ZAR; NAD mixing deferred). `GET /vendors/me/dashboard` endpoint `@Roles(UserRole.VENDOR)`, registered before `GET /:id` so NestJS matches the literal path first. 7 new Jest unit tests: revenue from decimal string, null aggregate → 0, null field → 0, status grouping, ZAR currency, product count passthrough, 404 on missing profile. api 72/72.

**Web (`apps/web`)**. `VendorsService.getDashboard()` → `GET /api/vendors/me/dashboard`. `VendorDashboard` component replaced placeholder with standalone, signals-based screen (no browser APIs — SSR-safe). State: `loading`, `error`, `dashboard` signals; `totalOrders` computed as sum of all status counts. Template: page header + "New Product" link → `../products`; 3 KPI cards (Total Products / Total Revenue / Total Orders); order-status breakdown chip grid (empty state when no orders); quick-link cards to Products and Orders. Styled to `docs/design/DESIGN.md` tokens throughout — no hardcoded colours. 13 Vitest specs in three top-level describe blocks (main, empty-state, error path). web 175/175.

#### Key decisions
- Counts order *line items* grouped by status (not distinct orders) — the read-model is for vendor fulfilment visibility; UI copy says "Order lines for your products" to match. Switching to `COUNT(DISTINCT o.id)` is a future follow-up.
- Currency hardcoded to `ZAR` for this slice — all `order_items` default to ZAR; NAD-priced lines are future work.
- Direct entity repo injection (not module import) to avoid circular dependencies, matching the AdminModule pattern from AP-9.

#### Follow-ups
- Switch `orderCountByStatus` to distinct-order count once the orders module has a real creation path (VP-4 / checkout card).
- NAD mixed-currency aggregation once NAD-priced products exist.
- `takeUntilDestroyed`/`toSignal` for the HTTP subscription (cosmetic — one-shot HTTP completes; no leak risk).

### Admin portal — order oversight + dashboard metrics — 2026-06-22 (card AP-10 / sw8qYBV1, PR #16)

**Branch:** `feat/sw8qYBV1-admin-orders-dashboard`
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/16
**Status:** In Review

#### What shipped
**Full-stack** — `@hb/shared` contracts + API read-only endpoints + Angular UI. No schema change, no migration.

**`@hb/shared`**: Three new contracts added to `libs/shared/src/contracts/`:
- `contracts/dashboard.ts`: `AdminDashboardDto` (pending vendor count, order totals, order counts by status, platform/vendor revenue grouped by currency), `CurrencyTotalDto`, `OrderStatusCountDto`.
- `contracts/order.ts`: `AdminOrderListItemDto`, `AdminOrderListDto`, `AdminOrderListQuery`.

**API (`apps/api/src/admin/`).** Read-only; no order mutations. `AdminModule` now registers `Order` / `OrderItem` / `Vendor` repos directly (same pattern as VP-2 to avoid circular imports). Two new endpoints:
- `GET /admin/dashboard` — returns pending-vendor count, total orders, order counts zero-filled across **every** `OrderStatus` (including unmatched states), and platform vs vendor revenue **grouped by currency** (ZAR and NAD never summed; the 1:1 peg is data, not assumption per [[Money & Currency Rules]]). Cancelled orders contribute zero revenue. Numeric strings coerced via `Number()`.
- `GET /admin/orders` — paginated list across all users/vendors (no ownership filter), filterable by `status` and `vendorId`. A vendor-filtered order keeps its full item set so `itemCount` and `vendorIds` remain accurate. Paginates in-memory at skeleton stage (DB-level pagination + query-level `vendorId` filter deferred to when order volume lands).

**Non-negotiable revenue-aggregation suite:** 24 new Jest unit tests incl. platform/vendor split, per-currency grouping (ZAR/NAD not summed), cancelled orders excluded, numeric-string coercion, zero-data empties. api 86/86.

**Web (`apps/web`).** New `core/api/admin-orders.service.ts` with `getDashboard()` and `listOrders(query)` (HttpParams built only from defined fields). Replaced placeholder Dashboard + Orders pages with real standalone, signals-based, SSR-safe screens (DESIGN.md tokens; mirrors `admin-users` / `admin-vendors` pattern).

**Dashboard:** metric cards for pending-vendors (CTA → `/admin/vendors`), total orders, orders-by-status, per-currency platform/vendor revenue. Empty state renders zeros / "No orders yet" — never an error.

**Orders:** paginated table (server-side `status` + `vendorId` filters), Prev/Next pagination, inline split list/detail panel. Zero-data state friendly, expected while orders module is skeleton. +66 Vitest specs. web 207/207.

#### Key decisions
- **AP-11 soft dependency only:** audit log is listed as "best shipped after" but not a blocker — AP-10 is read-only (no admin actions to audit). Real blockers were AP-6 + AP-9 (the `admin/` module foundation itself), both satisfied.
- Revenue = line-item GMV (`unitPrice * quantity`), grouped by currency, excluding cancelled orders. Shippingfees excluded (documented on `CurrencyTotalDto`).
- In-memory pagination at skeleton stage; DB-level + query-level filters follow when order volume arrives.

#### Follow-ups
- `GET /admin/orders`: DB-level pagination + query-level `vendorId` filter when checkout/order volume lands.
- Dedicated `/admin/orders/:id` order-detail route once the orders/checkout module is real.
- "Paid-only" revenue refinement (filter by payment status) once payments are real.

**Test/review outcome:** API 86/86 · Web 207/207 · lint clean · full build clean (only pre-existing SCSS warnings on `shop.scss` / `admin-catalog.scss`). Code review: **SHIP WITH NITS** — currency-symbol Record lookup + safe fallback, `CurrencyTotalDto` shipping-exclusion doc, test typo all addressed in-PR.

**This completes slice 10** of the admin-portal vertical sequence.
