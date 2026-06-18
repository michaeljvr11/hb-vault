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
