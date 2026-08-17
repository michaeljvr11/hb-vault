# Order State Machine

## Order lifecycle (`libs/shared/src/enums/order-status.ts`)

```
pending → confirmed → processing → handed_to_hb → shipped → delivered
   ↘ cancelled (allowed from pending/confirmed; later cancellation = TBD)
```

> **Note:** `handed_to_hb` is a new state (added 2026-06-18) — requires a new value in
> `libs/shared/src/enums/order-status.ts` and a TypeORM migration. It marks the moment a
> vendor has physically delivered their goods to HB for cross-border onward shipment.

## Transition rights (confirmed 2026-06-18)

| From | To | Who | Trigger |
|---|---|---|---|
| `pending` | `confirmed` | **admin** | payment authorized/paid — see [[Money & Currency Rules]] |
| `confirmed` | `processing` | **vendor** or admin | vendor begins picking/packing their lines |
| `processing` | `handed_to_hb` | **vendor** or admin | vendor signals goods delivered to HB logistics |
| `handed_to_hb` | `shipped` | **admin only** | HB books cross-border shipment — see [[Cross-Border & Customs]] |
| `shipped` | `delivered` | **admin only** | confirmed delivery to customer |
| `pending`/`confirmed` | `cancelled` | **admin** or customer | cancellation before fulfilment |

**Admin has full transition rights across all states.**
**Vendors are limited to their own order lines** (`order_items` where `vendorId` matches) and
may only move `confirmed → processing` and `processing → handed_to_hb`. They cannot
read/act on other vendors' lines or any platform-fulfilled lines.


## Admin override (confirmed 2026-08-16)

The transition table above governs the *normal* pipeline — vendor/customer/automatic
(payment) writes, all going through `ORDER_STATUS_TRANSITIONS`. Admin gets a second,
separate write path: an **unrestricted override** that can set an order to any
`OrderStatus` from any current status — backward, sideways, skipping stages, or leaving
a terminal state (`delivered`, `cancelled`). Rationale: orders can be falsely advanced
(wrong button, bad scan, vendor error) and admin needs a real correction path, not just
forward progression.

**Not the same code path as `ORDER_STATUS_TRANSITIONS`.** The override bypasses that
matrix entirely rather than extending it with reverse edges — the matrix expresses the
normal, happens-once forward flow; folding an emergency-correction case into the same
validation blurs a guarantee (`orders.service.ts`'s "single gateway" comment) that other
code currently relies on.

**Guardrails:**
- Admin-only (`@Roles(UserRole.ADMIN)`) — never exposed to vendor or customer.
- A **reason is required** on every override call (non-empty string, validated) — no
  reason, no write.
- Every override writes an **audit row** separate from normal order history: admin user
  id, order id, from-status, to-status, reason, timestamp. Additive audit data, not a
  replacement for anything existing.
- **Known risk, not solved here:** normal-pipeline side effects (e.g. the `order.paid`
  listener firing notification emails on `pending → confirmed`) can re-fire if an
  override re-enters that status. Whether override writes should suppress domain-event
  emission is an implementation decision, not resolved in this note — flag it for
  whoever picks up the card.

**Open implementation question:** a separate endpoint (e.g. `PATCH
/orders/:id/status-override`) vs. an unrestricted branch on the existing `PATCH
/orders/:id/status`. Recommend a **separate endpoint** — keeps the existing one's
"single validated gateway" guarantee intact for every other caller. Confirm before
implementation.

Tracked as Trello cards (API + admin UI) — see [[Product Reviews & Ratings]] for the
review-eligibility context that surfaced this gap.


**Notification-suppression question — resolved 2026-08-16.** The override UI exposes a
checkbox next to the required reason field: **"Send notification emails for this
change"**. Admin decides per-override, not a fixed policy. Default **unchecked** (no
notifications) — an override is typically a correction, not a real business event, and
silently emailing a customer "your order is confirmed!" because admin was fixing a
data-entry mistake would be worse than staying quiet. Admin opts in when the change is
a genuine status update that happens to be going through the override path (e.g.
legitimately marking `delivered` while the dedicated UI gap still exists).

Contract: `OrderStatusOverrideRequest` gains `sendNotifications: boolean` (required
field, no server-side default — the checkbox state is always sent explicitly). The
override write conditionally emits the normal domain event (`OrderEvents.PAID` etc.)
only when `sendNotifications` is true; the audit row records the override regardless.

## Coupled state machines

One order may have multiple shipments (mixed platform/vendor lines — see [[Listing Types & Vendor Rules]]). Order status aggregates over its shipments:

- Order is `shipped` when **all** its shipments are at least `in_transit` (DRAFT rule).
- Order is `delivered` when **all** shipments are `delivered` (DRAFT rule).
- Payment status is independent: an order can be `cancelled` with payment `refunded`.

## Rules for agents

- Every state transition goes through a service method that validates the from-state. No direct status writes from controllers.
- Every transition method gets a unit test (valid transitions + at least one rejected invalid transition).
- Status enums live in `@hb/shared` — never redefine.
- Vendor transition endpoints **must** verify `order_item.vendorId === user.vendorId` in the service layer before applying any status change.

## TBD (still open)

- Cancellation after `processing` (restocking? partial refunds?).
- Partial shipment / partial delivery handling (mixed-line orders where one vendor ships, another hasn't).


## Implementation notes

### Checkout one-shot — cart, order creation, state machine, isVerified gate (2026-07-07, cards #41 LT9UksZC / #43 zPeaeXhS / #42 XhSaiB4b / #15 6Rzyxd1R)

Shipped via [PR #27](https://github.com/michaeljvr11/hb-mono-repo/pull/27), branch `feat/fable-checkout-oneshot`. **One-time bundling of four cards into one branch** (mirroring the `fable-storefront-oneshot` precedent, PR #26) because they form one tightly-coupled purchase flow — cart shape, order-item shape and checkout UI had to land together.

**What shipped:**
- `OrderStatus.HANDED_TO_HB` is now real: added to `libs/shared/src/enums/order-status.ts` + migration `1783382400000-OrderStatusHandedToHb` (PG enum `ALTER TYPE ... ADD VALUE`; down rebuilds the type, falling `handed_to_hb` rows back to `processing`). Up/down verified on a fresh DB.
- **Cart API** (`apps/api/src/cart/`): `GET /cart`, `POST /cart/items`, `PATCH /cart/items/:id`, `DELETE /cart/items/:id`. The cart stores product refs + qty only; prices/stock/per-currency subtotals are read live from `products` at response time (cents math, ZAR/NAD never summed). Quantities clamp to live stock. Ownership is by construction — items resolve only through the caller's cart; foreign items are a plain 404.
- **Order creation** (`apps/api/src/orders/`): `POST /orders` builds the order from the server-side cart in one transaction — pessimistic `FOR UPDATE` row lock per product **in deterministic product-id order** (no oversell under concurrency, no deadlocks), stock re-checked + decremented, totals computed server-side from live prices, per-line snapshots (`unitPrice`/`productName`/`listingType`/`vendorId`), cart cleared. Insufficient stock → 409 + full rollback (no partial writes). Payment goes through the existing `PAYMENT_PROVIDER` stub port; a paid result transitions `pending → confirmed` through the same validation path as every other transition.
- **State machine enforced exactly as this note specifies**: `ORDER_STATUS_TRANSITIONS` map + `PATCH /orders/:id/status` as the single gateway (no direct status writes from controllers). Admin: full rights. Vendor: `confirmed → processing → handed_to_hb` only, service-layer check that the order holds at least one `order_items.vendorId` line of theirs (foreign orders 404, no existence leak). Customer: cancel own `pending`/`confirmed` only. Invalid from-states → 409 naming the exact transition.
- **isVerified order gate** (card #15, previously flagged as follow-up in [[Auth & Roles]]): `POST /orders` throws 403 "Verify your email before placing an order" when `user.isVerified` is false. Browse/cart remain open to unverified users.
- **Web**: auth-guarded `/cart` (line items, stock-clamped steppers, per-currency totals, empty state) and `/checkout` (typed reactive form + signals per the Claude Design checkout screen; confirmation state on success). All former "coming soon" add-to-cart placeholders (shop home / discovery / PDP / radial nav) now call the cart API; nav-bar + radial-nav badges show the real item count. The 403 verification error surfaces on checkout with a resend-verification action; 409 stock errors show the API's message and refresh the cart. Anonymous add-to-cart reuses the existing `/login?returnUrl` flow untouched.

**Key decisions:**
- **One order = one currency.** A cart mixing ZAR and NAD items is rejected at order creation (400) and blocked up front in the checkout UI — this note's sibling [[Money & Currency Rules]] says the two are never summed, and the `orders` schema carries a single `currency`. Splitting a mixed cart into two orders automatically was considered and deferred (payment/shipping would also split).
- **Shipping fee is explicitly 0.00** — the cross-border fee structure is still an open TBD in [[HB Domain Model]]; the `shippingTotal` column and UI row exist, honestly zero until priced.
- **Locking strategy**: pessimistic per-row `FOR UPDATE` (with `loadEagerRelations: false` — Postgres refuses `FOR UPDATE` on the nullable side of the image/category joins), products locked in sorted-id order to prevent deadlocks between concurrent checkouts sharing products.
- **Payment failure leaves the order `pending`** with stock decremented — unreachable with the deterministic stub; the retry/stock-release flow belongs to the future real-provider card.
- **Vendor transitions are order-level, not line-level**, matching the current single-status schema; partial/mixed-line fulfilment remains the TBD already listed above.

**Tests:** API 235/235 (Jest; +16 cart, +33 orders: money math, snapshots, stock/locks, rollback-on-conflict, verification gate, full valid+invalid transition matrix, per-role rights, ownership). Web 428/428 (Vitest; new CartService/cart-page/checkout suites incl. guard wiring and distinct 403-verify vs 409-stock surfacing). Build + lint clean; migrations run/revert/run verified.

**Follow-ups:**
- Real payment provider (Payfast/Paystack TBD) → replace stub adapter, add webhook-driven confirm + failure/stock-release path.
- Cross-border shipping fee rules → price `shippingTotal`, wire `SHIPPING_PROVIDER` to book a shipment on `handed_to_hb → shipped`.
- Customer "My Orders" page (API `GET /orders` already returns full DTOs; radial-nav "My Orders" is still a coming-soon snackbar).
- Vendor/admin UI actions for the new transitions (`PATCH /orders/:id/status` exists; portals don't call it yet).
- Mixed-currency cart checkout UX (auto-split into per-currency orders?) — ask a human.
### Vendor orders view + fulfilment read endpoint — 2026-07-17 (card VP-4 / sKd1xhQl)

Shipped via [PR (pending push)](https://github.com/michaeljvr11/hb-mono-repo), branch `feat/sKd1xhQl-vendor-orders-fulfilment`. **Scope discovery during CONTEXT:** the card's stated fulfilment-transition backend logic (confirmed→processing, processing→handed_to_hb, new enum + migration, 5 unit tests) was **already shipped** as part of the checkout one-shot (PR #27, 2026-07-07, documented above). This card's actual remaining work was narrower — a read API + vendor-scoped UI for what was already wired server-side.

**What shipped:**
- **`@hb/shared`**: `VendorOrderLineDto` added to `libs/shared/src/contracts/order.ts` — read-model interface (`id`, `orderId`, `orderStatus`, `orderCreatedAt`, `productName`, `unitPrice`, `currency`, `quantity`). **Deliberately did NOT add a separate `VendorFulfilmentUpdateDto`** — the card asked for it, but status changes reuse the already-shipped `PATCH /orders/:id/status` + `UpdateOrderStatusRequest` (which already validates vendor ownership and allowed transitions); adding a second DTO for the same endpoint would have violated the repo's "never duplicate DTOs" rule.
- **API** (`apps/api/src/orders/`): new `GET /orders/vendor` endpoint on `OrdersController` (declared before `GET /:id` for route-precedence), backed by `OrdersService.findAllForVendor(userId)` — resolves the caller's vendor row via the injected `vendorsRepository` (404 if the caller has no vendor row), then queries the `OrderItem` repository filtered by `vendorId`, mapped to `VendorOrderLineDto[]`. Four new unit tests in `apps/api/src/orders/orders.service.spec.ts`: ownership filter, 404-no-vendor, empty-array, DTO field mapping. api 26 suites / 347 tests / all pass.
- **Web** (`apps/web/src/app/features/vendor/pages/vendor-orders/`): replaced "Coming soon" placeholder with a real screen. New standalone, signals-based component (`.ts`/`.html`/`.scss`/`.spec.ts`) plus helper file `vendor-order-transitions.ts` (`vendorActionsFor(status)` / `vendorActionLabel(target)` — mirrors admin-vendors pattern to gate action buttons per order-line status). Extended `OrdersService` with `listForVendor()` and `updateStatus(id, status)`. State signals: `lines`/`loading`/`error`/`pendingId` (double-submit guard)/`actionError`; empty state renders as a friendly message (expected pre-checkout state), not an error; failed status-update shows inline error without discarding the list. Styled to `docs/design/DESIGN.md` tokens, mirroring `vendor-dashboard`/`admin-orders` (no new Claude Design export). 25 new Vitest specs. web 43 files / 471 tests / all pass.

**Key decision:** reused the existing `PATCH /orders/:id/status` endpoint instead of adding a duplicate fulfilment-update DTO. The endpoint already validated vendor ownership and allowed transitions in the service layer; a second DTO would have been redundant contract duplication.

**Tests/review outcome:** api 26 suites / 347 tests · web 43 files / 471 tests · lint clean · SSR build clean (no new warnings). Code review: **SHIP**, zero FAIL items. One non-blocking nit (not addressed, low priority): currency-symbol display in the vendor-orders HTML uses binary ZAR/NAD ternary rather than a full `CurrencyCode` map — acceptable for v1 (two-currency platform only), flagged as a follow-up if a third currency is ever added.

**Follow-ups:** the vendor-portal vertical slice (slice 4 in [[Vendor & Admin Portals]] — "Vendor orders & fulfilment") is now functionally complete for v1 (list + transitions). Currency-symbol ternary → full map if multi-currency support lands.

### Admin order-status override (emergency correction path) — 2026-08-17 (cards 5Ygendw9 / m0t7PTgP, branch `feat/5Ygendw9-admin-order-status-override`)

Full-stack API + web UI for admin-only, unrestricted order-status correction. Rationale: orders can be falsely advanced (wrong button, bad scan, vendor error) and admin needs an emergency-correction path outside the normal `ORDER_STATUS_TRANSITIONS` matrix.

**What shipped:**
- **`@hb/shared` contracts** (`libs/shared/src/contracts/order.ts`): new `OrderStatusOverrideRequest { status: OrderStatus; reason: string; sendNotifications: boolean }` and `OrderStatusOverrideAuditDto { id, orderId, adminUserId, adminEmail?, fromStatus, toStatus, reason, sendNotifications, createdAt }`.
- **API** (`apps/api/src/orders/`): new admin-only endpoints `PATCH /orders/:id/status-override` and `GET /orders/:id/status-overrides` (both `@Roles(UserRole.ADMIN)`). New `OrdersService.overrideStatus()` method deliberately bypasses `ORDER_STATUS_TRANSITIONS` — any status to any status, including terminal-state entry/exit. Rejects a no-op override (`dto.status === order.status`) with 409 before any write. Wraps the status save + conditional `deliveredAt` stamp (reusing atomic `UPDATE ... WHERE deliveredAt IS NULL` pattern) + audit-row insert in one DB transaction. Emits `OrderEvents.PAID` only when `sendNotifications === true` AND target status is `CONFIRMED` (the only domain event that exists today for any order status); no notification for other target statuses. Audit row written unconditionally on every successful override. New `order_status_overrides` table (migration `1787097600000-OrderStatusOverrides.ts`, FKs to `orders.id`/`users.id` `ON DELETE RESTRICT`) — separate from the generic `audit_logs` table. `findOverridesForOrder()` 404s on unknown order, joins `users` to populate `adminEmail` on each row.
- **Web** (`apps/web/src/app/features/admin/pages/admin-orders/`): new override control on the existing split list/detail admin-orders panel — any-status select (all `OrderStatus` values, not just forward-reachable), required reason textarea (no-op/whitespace rejected, max-length matching DB varchar(2000)), "Send notification emails for this change" checkbox (default unchecked), inline two-step confirmation (copy branches on checkbox state). Audit history list renders prior overrides (who/when/from→to/reason/notifications sent) for the selected order. New calls added to `OrdersService` (not `AdminOrdersService`; endpoints live on `OrdersController`).

**Key decisions:**
- **Separate endpoint** (`PATCH /orders/:id/status-override` vs. extending `/status`): preserves the existing "single validated gateway" guarantee for the normal pipeline. The override path is deliberately unrestricted (no `ORDER_STATUS_TRANSITIONS` check).
- **`OrderEvents.PAID` scoping**: notification emission on override scoped to the one event that already exists (fired only for `CONFIRMED` target). Overriding into any other status with notifications checked currently sends no email — no listener wired for those transitions yet. Marked as a follow-up if admin needs notification coverage for other override targets later.
- **No-op-override rejection**: 409 before any write/audit/event — discovered during code review, prevents silent re-fire of `CONFIRMED` notification fan-out on a "just add a reason and confirm" click when form defaults to current status.
- **Dedicated `order_status_overrides` table**, not generic `audit_logs` — every row always references exactly one order and one admin, so proper FKs (`ON DELETE RESTRICT`) made sense here in a way they don't for the polymorphic `audit_logs` table.
- **`adminEmail` populated via query-time join**, not denormalized column — kept migration/entity minimal.
- **Bundled as one PR, not two**: API and web cards shared the `OrderStatusOverrideRequest`/`OrderStatusOverrideAuditDto` contract one-for-one and the web card depended on the API card; mirrors PR #26/#27/#28 precedent.

**Review process:** Two rounds of real findings (non-fatal bugs in first pass): weak reason validation (whitespace-only accepted, no max-length vs varchar(2000) column), three non-atomic writes (status/deliveredAt/audit — now wrapped in transaction), no-op-override notification re-fire (now rejected with 409), missing tests for `@Roles(ADMIN)` gate and DTO validation, stale-response race in web detail panel (switching selected order mid-flight could render one order's audit history under another, or leave double-submit lock stuck). All fixed and re-reviewed; a third final pass caught one residual regression in the race-condition fix itself (stale-response guard released double-submit lock too late on the stale branch, permanently soft-locking Confirm after mid-flight order switch) — fixed and verified. Final verdict: **SHIP**.

**Tests:** API 54 suites / 741 tests (lint clean). Web 69 files / 880 tests. Full `npm run build` clean (pre-existing non-fatal SCSS budget warnings on 2 unrelated files; `admin-orders.scss` now also trips the same warning by ~2.7kB over budget — flagged as follow-up). Migration `OrderStatusOverrides1787097600000` verified up/down/up against live dev Postgres.

**Follow-ups:** SCSS budget extraction (admin pages sharing `.action-bar`/`.action-btn`/spinner CSS blocks); notification coverage for other override targets (future card if needed).
