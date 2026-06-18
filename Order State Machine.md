# Order State Machine

## Order lifecycle (`libs/shared/src/enums/order-status.ts`)

```
pending → confirmed → processing → shipped → delivered
   ↘ cancelled (allowed from pending/confirmed; later cancellation = TBD)
```

## DRAFT transition rules (proposed — confirm with a human before enforcing)

| From | To | Trigger |
|---|---|---|
| `pending` | `confirmed` | payment authorized/paid — see [[Money & Currency Rules]] |
| `confirmed` | `processing` | fulfilment started (platform or vendor) |
| `processing` | `shipped` | shipment booked + in transit — see [[Cross-Border & Customs]] |
| `shipped` | `delivered` | shipment delivered |
| `pending`/`confirmed` | `cancelled` | customer or admin cancels |

## Coupled state machines

One order may have multiple shipments (mixed platform/vendor lines — see [[Listing Types & Vendor Rules]]). Order status aggregates over its shipments:

- Order is `shipped` when **all** its shipments are at least `in_transit` (DRAFT rule).
- Order is `delivered` when **all** shipments are `delivered` (DRAFT rule).
- Payment status is independent: an order can be `cancelled` with payment `refunded`.

## Rules for agents

- Every state transition goes through a service method that validates the from-state. No direct status writes from controllers.
- Every transition method gets a unit test (valid transitions + at least one rejected invalid transition).
- Status enums live in `@hb/shared` — never redefine.

## TBD (ask a human)

- Cancellation after `processing` (restocking? partial refunds?).
- Partial shipment / partial delivery handling.
- Who can trigger which transition (customer vs vendor vs admin).
