# Order Tracking

Status: **Spec'd 2026-10-05**. Cards are in Trello "To Do":
- OT-1 `EUy8sgIY` — API foundation (blocks the rest)
- OT-2 `Z3gQhcTv` — admin Update tracking panel
- OT-3 `HD13Bk1b` — customer tracker + deep link + copy
- OT-4 `zSqz4wBg` — milestone emails (ship with or after OT-3, because the email links to its route)
Related: [[Order State Machine]] · [[Cross-Border & Customs]] ·
[[Transactional Email & Order Notifications]] · [[Vendor & Admin Portals]] ·
[[Customer Profile]] · [[Vendor Earnings & Commission]] · [[Legal & Compliance Readiness]]

## Problem

Customers should feel informed and reassured while an order travels ZA → NA. Admins
need a simple way to move an order through its delivery stages. No courier
integration exists yet, so for now an admin drives every stage by hand. The design
must still let a real courier adapter feed the same timeline later.

## Current state (verified in code, 2026-10-05)

- **Customer:** `/profile/orders` (`profile-orders.ts`) shows a single status badge.
  The label is a humanised enum value, so one stage renders as "Handed to hb". There is
  no timeline, no per-stage date, no estimated delivery date and no tracking reference.
  The detail view is an in-component selection, so it has **no deep-linkable URL**.
- **Email:** the only customer order email is the TE-5 confirmation on `order.paid`.
  `OrderEvents` has a single event (`PAID`). Nothing fires for shipped or delivered.
- **Admin:** the admin-orders page **never calls `PATCH /orders/:id/status`**. Its only
  status control is the override escape hatch ([[Order State Machine]] § Admin override).
  That control needs a reason, defaults to sending no email, and only emails for a
  `confirmed` target. Admin has been using the correction tool to progress real orders.
- **No status history.** `updateStatus` stamps nothing except `deliveredAt`. Only
  overrides write audit rows (`order_status_overrides`).
- **Unused groundwork:** the `shipments` table, the `ShipmentStatus` enum
  (`pending → booked → in_transit → at_border → customs_cleared → out_for_delivery →
  delivered`, plus `failed`), `trackingReference`, `customsReference`, and the
  `SHIPPING_PROVIDER` port with `StubShippingProvider`. **Nothing reads or writes any of
  it**, and no endpoint exposes it.
- **Public copy admits the gap:** `export-customs.html` §"Tracking your order through
  customs" says there is no tracking screen and tells customers to ask support.

## Owner decisions (confirmed 2026-10-05)

| Question | Decision |
|---|---|
| Customer-facing stages | **Simplified:** Confirmed → Preparing → Shipped → At the border → Out for delivery → Delivered |
| Domestic ZA orders | Skip **At the border** (stepper shows 5 steps) |
| Channel | **Email only** for now. SMS/WhatsApp can hang off the same event later |
| Which updates email the customer | The **Shipped, At the border, Out for delivery and Delivered** milestones default ON. Every admin update has a **"Notify customer" toggle** that can override the default either way |
| Mixed-vendor orders | **One tracker per order.** H&B consolidates after hand-off |
| Admin notes + estimated delivery date | **Visible to the customer** |
| Customs reference | **Internal only.** Never in a customer DTO or email |
| Vendor transitions | Vendors keep `processing` / `handed_to_hb`. The customer sees **both** as "Preparing" |
| Admin override | Stays **correction-only**. It never sends tracking emails |

## Business rules

### R1 — Customer stage is derived, never stored

`TrackingStage` is computed from `(order.status, shipment.status)` by one API function,
covered by a unit test for every row below:

| Order status | Shipment status | `TrackingStage` |
|---|---|---|
| `pending` | — | `pending` ("Awaiting payment". Shown as a banner, not a step) |
| `confirmed` | — | `confirmed` |
| `processing`, `handed_to_hb` | any / none | `preparing` |
| `shipped` | none, `pending`, `booked`, `in_transit` | `shipped` |
| `shipped` | `at_border`, `customs_cleared` | `at_border` |
| `shipped` | `out_for_delivery` | `out_for_delivery` |
| `delivered` | any / none | `delivered` |
| `cancelled` | any / none | `cancelled` (a banner replaces the stepper) |

`failed` is out of scope for v1 (see Out of scope). It must not be reachable from the
admin panel.

### R2 — One shipment per order (v1)

- A tracking update lazily creates the order's single `shipments` row (`provider =
  'manual'`, `fromCountry`/`toCountry` copied from the order). A **unique index on
  `shipments.orderId`** enforces one shipment per order and keeps concurrent creation
  race-safe.
- Dropping that index is part of any future multi-parcel card. This replaces the DRAFT
  aggregation rule in [[Order State Machine]] § Coupled state machines for v1. With one
  shipment, "all shipments" means "the shipment".

### R3 — Admin tracking targets and coupled transitions

The admin chooses a `TrackingTarget`, or leaves it out for a note-only update. The
service applies the coupled order and shipment writes **in one DB transaction**.
Order writes go through the existing `assertValidTransition` gateway. They never
write status directly.

| `TrackingTarget` | Order write | Shipment write | Customer sees | Email default |
|---|---|---|---|---|
| `processing` | `confirmed → processing` | — | Preparing | OFF |
| `handed_to_hb` | `processing → handed_to_hb` | — | Preparing | OFF |
| `shipped` | `handed_to_hb → shipped` | → `in_transit` | Shipped | **ON** |
| `at_border` | — (order is `shipped`) | `in_transit → at_border` | At the border | **ON** |
| `customs_cleared` | — | `at_border → customs_cleared` | At the border (an activity entry "Cleared customs") | OFF |
| `out_for_delivery` | — | `customs_cleared → out_for_delivery` (cross-border) / `in_transit → out_for_delivery` (domestic) | Out for delivery | **ON** |
| `delivered` | `shipped → delivered` (stamps `deliveredAt`) | → `delivered` | Delivered | **ON** |
| *(none)* — note only | — | — | Activity entry under the current step | OFF |

- **Domestic** means `order.originCountry === order.destinationCountry`. On a domestic
  order, `at_border` and `customs_cleared` are rejected with 409.
- **Customs rule** (from [[Cross-Border & Customs]]): `customs_cleared` requires a
  `customsReference`, either already stored or supplied in the same request. This
  enforces "required before a shipment can leave `at_border`".
- **`deliveredAt`** is stamped with the same race-safe conditional `UPDATE … WHERE
  "deliveredAt" IS NULL` that `updateStatus` uses today. It is a payout-clock anchor
  ([[Vendor Earnings & Commission]]), so it needs a test.
- **Re-shipping after a correction:** if an override moved the order back out of
  `shipped`, the `shipped` target reuses the existing shipment row and resets it to
  `in_transit`.
- Out-of-order targets are rejected with 409 and a message naming the from/to pair.
- The API returns the valid **`nextTargets`**, each with its `notifyByDefault`, so the
  web never re-implements these rules.

### R4 — Every order status change appends a timeline event

A new append-only `order_tracking_events` table is the single timeline. Every write
path appends one row:

| Path | `source` | `visibleToCustomer` | `notifiedCustomer` |
|---|---|---|---|
| Payment confirm (`capturePayment`, `pending → confirmed`) | `system` | true | false (TE-5 confirmation already covers it) |
| `PATCH /orders/:id/status` by a vendor | `vendor` | true | false |
| `PATCH /orders/:id/status` by an admin or customer (e.g. cancel) | `admin` / `customer` | true | false |
| Admin tracking update (R3) | `admin` | true | the request's `notifyCustomer` |
| Status override | `override` | **false** | false |

- Each row snapshots `orderStatus` and `shipmentStatus` (nullable) after the write. It
  also stores an optional `note`, `actorUserId` (null for `system`) and `occurredAt`.
  The derived stage comes from the snapshot through R1.
- Hiding override rows keeps corrections out of the customer's activity list. The
  stepper always reflects the current state.
- **No backfill.** Existing orders have no events, and the stepper renders their steps
  without dates. There is no live deployment yet (2026-08-26), so only dev data is
  affected.
- **Step timestamps:** a step's `reachedAt` is the `occurredAt` of the earliest visible
  event at that stage that comes after the most recent visible event at an earlier
  stage. This means a re-entered stage after a correction shows its latest arrival.

### R5 — Customer visibility

- The customer DTO and email carry: `TrackingStage`, steps with `reachedAt`,
  `estimatedDeliveryDate`, `carrierName`, `trackingReference`, and visible event notes.
- They **never** carry `customsReference`, the event `source`, the actor, override rows
  or `notifiedCustomer`.
- The customer may read only their own order. Other users get 404, with no existence
  leak, matching `findOneForUser`.

### R6 — Milestone emails

- After commit, an admin tracking update with `notifyCustomer: true` emits
  `OrderEvents.TRACKING_UPDATED { orderId, trackingEventId }` on a best-effort basis.
  This follows the same pattern and the same no-retry caveat as `order.paid`.
- A listener sends the customer one email built with `renderEmail`. It contains:
  - a stage headline, e.g. "Your order #ab12cd34 is at the border";
  - the admin note, if there is one;
  - the estimated delivery date, if there is one;
  - the carrier and tracking reference, if there are any;
  - a `link` block to `${APP_WEB_URL}/profile/orders/<id>`.
- It **never** includes the customs reference or money figures beyond what TE-5
  already shows.
- The email is sent only when the admin's toggle is on. Vendor, system, customer and
  override writes never send tracking email.
- A note-only update with the toggle on sends a "Update on your order" email.
- The **override** path is unchanged: `sendNotifications` still only re-fires `order.paid`
  for a `confirmed` target. Override help text should say "use Update tracking for real
  progress".

## `@hb/shared` contract impact

All additions; nothing existing changes shape except `OrderDto` gaining one field.

- `enums/tracking-stage.ts` — `TrackingStage`: `pending | confirmed | preparing |
  shipped | at_border | out_for_delivery | delivered | cancelled`.
- `enums/tracking-target.ts` — `TrackingTarget`: `processing | handed_to_hb | shipped |
  at_border | customs_cleared | out_for_delivery | delivered`.
- `enums/tracking-event-source.ts` — `TrackingEventSource`: `system | vendor | customer
  | admin | override | courier` (`courier` is reserved for the future adapter).
- `contracts/tracking.ts`:
  - `TrackingStepDto { stage: TrackingStage; state: 'done' | 'current' | 'upcoming';
    reachedAt?: string }`
  - `TrackingEventDto { id; stage: TrackingStage; shipmentStatus?: ShipmentStatus;
    note?: string; occurredAt: string }` (customer-safe)
  - `OrderTrackingDto { orderId; stage: TrackingStage; crossBorder: boolean;
    steps: TrackingStepDto[]; estimatedDeliveryDate?: string /* YYYY-MM-DD */;
    carrierName?; trackingReference?; events: TrackingEventDto[] }`. Events are newest
    first and visible only.
  - `AdminTrackingEventDto extends TrackingEventDto { orderStatus: OrderStatus; source:
    TrackingEventSource; actorEmail?; visibleToCustomer: boolean; notifiedCustomer:
    boolean }`
  - `TrackingNextTargetDto { target: TrackingTarget; notifyByDefault: boolean }`
  - `AdminOrderTrackingDto` — every `OrderTrackingDto` field, plus `shipmentStatus?`,
    `customsReference?`, `nextTargets: TrackingNextTargetDto[]`, and
    `events: AdminTrackingEventDto[]`. Events include hidden rows.
  - `AddTrackingUpdateRequest { target?: TrackingTarget; note?: string;
    estimatedDeliveryDate?: string; carrierName?: string; trackingReference?: string;
    customsReference?: string; notifyCustomer: boolean }`. `notifyCustomer` is required
    with no server default, following the override `sendNotifications` precedent: the web
    pre-fills it from `notifyByDefault`. A request with neither `target` nor `note` is
    invalid (400). Length limits: `note` ≤ 1000; `carrierName` / `trackingReference` /
    `customsReference` ≤ 100. `estimatedDeliveryDate` must be an ISO date.
- `OrderDto` gains `trackingStage: TrackingStage`. This lets list badges use
  customer-friendly labels without a second request.

### Endpoints

| Method + path | Who | Body / response |
|---|---|---|
| `GET /orders/:id/tracking` | owning customer (and admin) | → `OrderTrackingDto` |
| `GET /admin/orders/:id/tracking` | admin | → `AdminOrderTrackingDto` |
| `POST /admin/orders/:id/tracking` | admin | `AddTrackingUpdateRequest` → `AdminOrderTrackingDto` |

### Schema (one migration)

- New `tracking_event_source` PG enum. New `order_tracking_events` table: `id`,
  `orderId` (FK `orders`, `ON DELETE CASCADE`), `orderStatus` (`order_status`),
  `shipmentStatus` (`shipment_status`, nullable), `source`, `note` (varchar 1000,
  nullable), `visibleToCustomer` (bool), `notifiedCustomer` (bool), `actorUserId` (FK
  `users`, nullable, `ON DELETE SET NULL`) and `occurredAt` (timestamptz). Plus an
  index on `(orderId, occurredAt)`.
- `shipments` gains `estimatedDeliveryDate` (`date`, nullable) and `carrierName`
  (varchar 100, nullable). It also gets a **unique index on `orderId`** (R2).
- Down must be symmetric. `synchronize` stays off.

## UI

- **Admin** (`admin-orders` detail): a new **Update tracking** section sits above the
  override section. It contains:
  - a read-only stepper;
  - a radio/select of `nextTargets`, plus a "Note only" option;
  - an optional note;
  - an estimated delivery date picker, prefilled from the current value;
  - carrier name and tracking reference fields;
  - a customs reference field, shown only on cross-border orders and marked "internal";
  - a **"Notify customer by email"** checkbox, prefilled from `notifyByDefault` and
    re-prefilled when the target changes;
  - a submit button guarded against double-submit;
  - the full timeline, including hidden override rows with their badges.

  Two small fixes come with it: add a "Handed to HB" filter tab, and correct the
  humanised label to "Handed to HB". The override help text points real progress to
  this panel.
- **Customer** (`profile-orders`):
  - A deep-linkable `/profile/orders/:id` route. The email links there, and the auth
    guard handles login via `returnUrl`.
  - A visual stepper: 6 steps cross-border, 5 domestic. Each step shows done/current/
    upcoming and its date.
  - The estimated delivery date, carrier and tracking reference, plus an "Updates" list
    of notes, newest first.
  - A pending banner ("Awaiting payment") and a cancelled banner.
  - The list badges use the `trackingStage` label.

  Styling follows the `docs/design/DESIGN.md` tokens. There is no Claude Design export
  for this screen; if the card wants one, it pulls the design via DesignSync first.
- **Copy:** `export-customs.html` §"Tracking your order through customs" now says
  customers can follow their order, including when it reaches the border, from their
  account. It no longer mentions asking support. The customs reference stays internal,
  so the copy must not promise it.

## Courier-ready seam

When a courier is chosen, its adapter (`SHIPPING_PROVIDER`) writes shipment statuses
through the **same** tracking service with `source = 'courier'`. That produces the same
events, the same R1 derivation and the same `TRACKING_UPDATED` emails. No UI changes
are needed. This spec does not touch the port.

## Out of scope (v1)

- **Delivery failure / return to sender** (`ShipmentStatus.failed`) and any refund or
  restock flow. These are tied to the open cancellation-after-processing TBD in
  [[Order State Machine]].
- Multiple shipments per order, and partial shipment or delivery.
- SMS / WhatsApp / push notifications, and per-user notification preferences.
- Vendor-facing tracking view or vendor emails for tracking stages.
- A public (logged-out) tracking page or tracking-number lookup.
- A real courier adapter and automatic polling.
- Backfilling timeline events for existing orders.

## Open questions (ask a human)

1. **Estimated delivery date fallback:** before an admin sets a date, should the customer
   see nothing (the default in this spec) or a computed fallback? A fallback could come
   from the SVO-3 lead-time window plus the shipping-quote `estimatedDays`. **Default:
   nothing.** Revisit once SVO-3 ships.
2. **Courier tracking reference visible to customers:** yes by default, so customers can
   check with the courier themselves. Confirm.
3. **Cancelled after shipping:** the state machine forbids it today. This spec does not
   change that.
