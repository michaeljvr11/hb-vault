# Order Tracking

Status: **Spec'd 2026-10-05**. OT-1/2/3 implemented on branch `feat/EUy8sgIY-order-tracking` (PR pending). OT-4/5/6 in To Do.
- OT-1 `EUy8sgIY` — API foundation (blocks the rest) ✓ implemented (PR pending)
- OT-2 `Z3gQhcTv` — admin Update tracking panel ✓ implemented (PR pending)
- OT-3 `HD13Bk1b` — customer tracker + deep link + copy ✓ implemented (PR pending)
- OT-4 `zSqz4wBg` — milestone emails (ship with or after OT-3, because the email links to its route)
- OT-5 `25OQwbvD` — vendor tracking view (after OT-1)
- OT-6 `qJrrBvaJ` — vendor "delivered" email (ship with or after OT-5, because the email links to its route)
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
| Courier details *(follow-up, same day)* | A courier partner is coming, but which one is unknown. The fields stay **generic**: courier name, tracking reference and tracking link. The admin decides whether to share them with the customer through a **"Share courier details with customer"** switch on the order. When it is on, they appear on the tracker and in every tracking email |
| Vendor-facing tracking *(follow-up, same day)* | **In scope.** Vendors get a read-only tracker for orders containing their lines (OT-5) |
| Vendor emails *(follow-up, same day)* | **Delivered only.** Each vendor on the order gets one email when it is delivered (R8) |
| Estimated delivery date fallback *(follow-up, same day)* | Until an admin sets an exact date, show a **route-based window from order confirmation**. Domestic (ZA→ZA or NA→NA): **7–14 days**. Cross-border: **14–28 days**, matching the published Shipping Policy; the owner first said 14–21, then chose 14–28 for consistency. Both windows are platform settings, not hard-coded (R9). A product-aware estimate is **for later** |
| SVO-3 relationship *(follow-up, same day)* | SVO-3's per-product setting means **door-to-door delivery time**, the same concept as R9. There is **one** set of platform defaults: R9's settings, created by OT-1. SVO-3 is re-scoped to add only the per-product override and to render PDP / product card / checkout from data. **Ship the OT cards first, then SVO-3** |

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

### R5 — Customer and vendor visibility

- The customer-safe `OrderTrackingDto` and the email carry: `TrackingStage`, steps
  with `reachedAt`, `estimatedDeliveryDate`, visible event notes, and courier details
  **only when shared** (R7).
- They **never** carry `customsReference`, the event `source`, the actor, override rows
  or `notifiedCustomer`.
- **Who may read `GET /orders/:id/tracking`:**
  - the ordering customer;
  - a vendor with at least one `order_items` line on the order (the same ownership check
    as `assertActorMayTransition`);
  - an admin.

  Anyone else gets 404, with no existence leak.
- **Vendors get the same customer-safe DTO**, so the R7 sharing switch applies to them
  too. They do not get the customer's address or contact details through this endpoint.
  The DTO carries none.

### R6 — Milestone emails

- After commit, an admin tracking update with `notifyCustomer: true` emits
  `OrderEvents.TRACKING_UPDATED { orderId, trackingEventId }` on a best-effort basis.
  This follows the same pattern and the same no-retry caveat as `order.paid`.
- A listener sends the customer one email built with `renderEmail`. It contains:
  - a stage headline, e.g. "Your order #ab12cd34 is at the border";
  - the admin note, if there is one;
  - the estimated delivery date, if there is one;
  - a courier block (carrier, tracking reference, and a "Track with <carrier>" `link`
    when there is a URL), **only when `shareCourierDetails` is true** (R7);
  - a `link` block to `${APP_WEB_URL}/profile/orders/<id>`.
- It **never** includes the customs reference or money figures beyond what TE-5
  already shows.
- The email is sent only when the admin's toggle is on. Vendor, system, customer and
  override writes never send tracking email.
- A note-only update with the toggle on sends a "Update on your order" email.
- The **override** path is unchanged: `sendNotifications` still only re-fires `order.paid`
  for a `confirmed` target. Override help text should say "use Update tracking for real
  progress".

### R7 — Courier details and the admin sharing switch

- The shipment row stores generic courier fields: `carrierName`, `trackingReference` and
  `trackingUrl` (the courier's own tracking page). They make no assumption about which
  courier is used.
- `trackingUrl` must be an absolute `https://` URL, ≤ 500 chars. It is rendered only as
  a link, never as raw HTML.
- `shareCourierDetails` (boolean, **default false**) is a persistent setting on the
  shipment that the admin controls:
  - When it is false, `OrderTrackingDto` **omits all three courier fields** on the
    server, and emails leave them out.
  - When it is true, the customer tracker shows them, with a "Track with <carrier>"
    button when there is a URL. Every tracking email from then on includes a courier
    block.
  - Admin DTOs always carry the fields and the switch.
- The request field is optional: omit it to leave the setting unchanged. The admin
  panel prefills the checkbox ON the first time courier details are entered, so the
  common case takes one click. The admin can still untick it.
- **Announcing courier details:** there is no dedicated "courier details added" email.
  The admin sends one by posting the update (or a note-only update) with "Notify
  customer" on, and the courier block rides along.
- **Courier-ready:** a future courier adapter fills the same three fields
  automatically. Sharing stays an admin decision.

### R8 — Vendor "delivered" email

- A new `OrderEvents.DELIVERED { orderId }` is emitted after commit **only when the
  conditional `deliveredAt` stamp actually writes a row**, i.e. the order's *first*
  delivery. It fires on the admin tracking path (`delivered` target) and on
  `PATCH /orders/:id/status`. It **never** fires on the override path, which stays
  correction-only. This de-duplicates by construction: re-entering `delivered` after a
  correction never re-emails, because `deliveredAt` is never cleared.
- The listener sends **one email per distinct vendor** with lines on the order. It
  resolves the address with the existing `VendorsService.resolveNotificationEmail` and
  skips with a warning when none is found. Platform-only lines get no email.
- Content:
  - that vendor's own lines only (name, size, quantity, unit price, the same shape as
    TE-4);
  - the delivered date;
  - one line about payout timing: eligible once the damage-claim window closes. Use
    `DAMAGE_CLAIM_WINDOW_HOURS`, never a literal 48 ([[Vendor Earnings & Commission]]);
  - a link to `${APP_WEB_URL}/vendor/orders/<orderId>` (OT-5).
- It **never** carries commission or net-earnings figures, other vendors' lines,
  customer contact or address, or the customs reference.
- The send is best-effort and isolated per vendor (the `safely()` shape), so one vendor's
  failure never blocks another's.
- This is independent of the admin's "Notify customer" toggle: the toggle controls
  customer email only.

### R9 — Estimated delivery date: exact date or default window

- If the admin has set `estimatedDeliveryDate`, that exact date is shown and emailed.
- Otherwise, `OrderTrackingDto.estimatedDeliveryWindow = { earliest, latest }`. These
  are the **order confirmation time** + the **route's** min / max days, as `YYYY-MM-DD`
  in Africa/Johannesburg (the same local day in Windhoek).
  - **Domestic** (`originCountry === destinationCountry`, i.e. ZA→ZA or NA→NA):
    `domesticDeliveryDaysMin` / `Max`, seeded **7 / 14**.
  - **Cross-border:** `crossBorderDeliveryDaysMin` / `Max`, seeded **14 / 28**.
- The confirmation time is the `occurredAt` of the order's `confirmed` tracking event,
  falling back to `createdAt` for legacy orders with no events. This matches the
  Shipping Policy wording "from order confirmation".
- The window is present only while the stage is `confirmed` … `out_for_delivery`, and
  never for `pending`, `delivered` or `cancelled`. Emails show the same value.
- The four settings live in `platform_settings` and are editable on the admin settings
  screen. Validation, per pair: integers, 1 ≤ min ≤ max ≤ 120. They are the **single
  set of platform-wide door-to-door defaults**. SVO-3 must reuse them, not add its own.
- The window is computed at read time from the current settings. Changing them shifts
  the estimate on open orders. That is acceptable for v1 because no estimate is promised
  at checkout today.
- **Running late:** if today is past `latest` and the order isn't delivered, the customer
  UI replaces the date range with reassuring copy, e.g. "Taking a little longer than
  usual — we'll update you here". An admin note or exact date overrides it.
- **Published copy** (today hard-coded "14–28 days" for every route):
  - OT-3 aligns it now with static text:
    - Shipping Policy § "How long delivery takes": domestic 7–14 days, cross-border
      14–28 days, both from order confirmation.
    - Checkout domestic banner branch gains "Typically 7–14 days". The existing comment
      about there being "no sourced domestic SLA" is now obsolete; the owner sourced it
      on 2026-10-05.
    - PDP door-stop detail covers both routes.
  - Re-scoped SVO-3 later swaps the checkout and PDP strings for data.
  - The Shipping Policy stays static legal copy. Whoever changes these settings away
    from the seeds updates the policy page in the same change.
- **Later, not carded:** once SVO-3 ships per-product door-to-door overrides, derive a
  product-aware tracker estimate from the order's lines. Use the **slowest** line's
  resolved window (product override → R9 route default), and keep R9's route default as
  the fallback.

## `@hb/shared` contract impact

All additions; nothing existing changes shape except `OrderDto` and
`VendorOrderLineDto` each gaining one field.

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
    estimatedDeliveryWindow?: { earliest: string; latest: string };
    carrierName?; trackingReference?; trackingUrl?; events: TrackingEventDto[] }`.
    Courier fields are present only when shared (R7). `estimatedDeliveryWindow` is
    present only when there is no exact date (R9). Events are newest first and
    visible only.
  - `AdminTrackingEventDto extends TrackingEventDto { orderStatus: OrderStatus; source:
    TrackingEventSource; actorEmail?; visibleToCustomer: boolean; notifiedCustomer:
    boolean }`
  - `TrackingNextTargetDto { target: TrackingTarget; notifyByDefault: boolean }`
  - `AdminOrderTrackingDto` — every `OrderTrackingDto` field (courier fields always
    present when set), plus `shipmentStatus?`, `customsReference?`,
    `shareCourierDetails: boolean`, `nextTargets: TrackingNextTargetDto[]`, and
    `events: AdminTrackingEventDto[]`. Events include hidden rows.
  - `AddTrackingUpdateRequest { target?: TrackingTarget; note?: string;
    estimatedDeliveryDate?: string; carrierName?: string; trackingReference?: string;
    trackingUrl?: string; customsReference?: string; shareCourierDetails?: boolean;
    notifyCustomer: boolean }`. `notifyCustomer` is required
    with no server default, following the override `sendNotifications` precedent: the web
    pre-fills it from `notifyByDefault`. A request with neither `target` nor `note` is
    invalid (400). Length limits: `note` ≤ 1000; `carrierName` / `trackingReference` /
    `customsReference` ≤ 100. `trackingUrl` must be `https://` and ≤ 500.
    `estimatedDeliveryDate` must be an ISO date.
- `OrderDto` gains `trackingStage: TrackingStage`. This lets list badges use
  customer-friendly labels without a second request.
- `VendorOrderLineDto` gains `trackingStage: TrackingStage` (OT-5), for the same reason
  on the vendor orders list.
- `PlatformSettingsDto` and `UpdatePlatformSettingsRequest` gain
  `domesticDeliveryDaysMin` / `domesticDeliveryDaysMax` and `crossBorderDeliveryDaysMin`
  / `crossBorderDeliveryDaysMax` (R9).
- The API domain events `OrderEvents` gain `TRACKING_UPDATED` (R6) and `DELIVERED` (R8).
  These are API-internal, not `@hb/shared`.

### Endpoints

| Method + path | Who | Body / response |
|---|---|---|
| `GET /orders/:id/tracking` | owning customer, vendor with a line on the order (OT-5), admin | → `OrderTrackingDto` |
| `GET /admin/orders/:id/tracking` | admin | → `AdminOrderTrackingDto` |
| `POST /admin/orders/:id/tracking` | admin | `AddTrackingUpdateRequest` → `AdminOrderTrackingDto` |

### Schema (one migration)

- New `tracking_event_source` PG enum. New `order_tracking_events` table: `id`,
  `orderId` (FK `orders`, `ON DELETE CASCADE`), `orderStatus` (`order_status`),
  `shipmentStatus` (`shipment_status`, nullable), `source`, `note` (varchar 1000,
  nullable), `visibleToCustomer` (bool), `notifiedCustomer` (bool), `actorUserId` (FK
  `users`, nullable, `ON DELETE SET NULL`) and `occurredAt` (timestamptz). Plus an
  index on `(orderId, occurredAt)`.
- `shipments` gains `estimatedDeliveryDate` (`date`, nullable), `carrierName`
  (varchar 100, nullable), `trackingUrl` (varchar 500, nullable) and
  `shareCourierDetails` (boolean, not null, default false). It also gets a **unique
  index on `orderId`** (R2).
- `platform_settings` gains `domesticDeliveryDaysMin` / `Max` (int, not null, defaults
  7 / 14) and `crossBorderDeliveryDaysMin` / `Max` (int, not null, defaults 14 / 28) (R9).
- Down must be symmetric. `synchronize` stays off.

## UI

- **Admin** (`admin-orders` detail): a new **Update tracking** section sits above the
  override section. It contains:
  - a read-only stepper;
  - a radio/select of `nextTargets`, plus a "Note only" option;
  - an optional note;
  - an estimated delivery date picker, prefilled from the current value;
  - a **Courier details** group: carrier name, tracking reference, a tracking link,
    and a **"Share courier details with customer"** checkbox (R7);
  - a customs reference field, shown only on cross-border orders and marked "internal";
  - a **"Notify customer by email"** checkbox, prefilled from `notifyByDefault` and
    re-prefilled when the target changes;
  - a submit button guarded against double-submit;
  - the full timeline, including hidden override rows with their badges.

  Admin **settings** gains a "Default delivery estimate (days)" group with two min/max
  pairs: Domestic and Cross-border (R9).

  Two small fixes come with it: add a "Handed to HB" filter tab, and correct the
  humanised label to "Handed to HB". The override help text points real progress to
  this panel.
- **Customer** (`profile-orders`):
  - A deep-linkable `/profile/orders/:id` route. The email links there, and the auth
    guard handles login via `returnUrl`.
  - A visual stepper: 6 steps cross-border, 5 domestic. Each step shows done/current/
    upcoming and its date.
  - The estimated delivery date: the exact date if set, otherwise the R9 window (e.g.
    "Estimated delivery: 19 – 26 Oct"), otherwise the running-late copy. Also an
    "Updates" list of notes, newest first. When
    courier details are shared, it also shows the carrier, the tracking reference and a
    "Track with <carrier>" button (opens in a new tab, `rel="noopener noreferrer"`).
  - A pending banner ("Awaiting payment") and a cancelled banner.
  - The list badges use the `trackingStage` label.

  Styling follows the `docs/design/DESIGN.md` tokens. There is no Claude Design export
  for this screen; if the card wants one, it pulls the design via DesignSync first.
- **Vendor** (`vendor-orders`, OT-5):
  - The list gains a stage badge (`trackingStage`) and a "Track" action that opens a
    deep-linkable `/vendor/orders/:orderId` detail.
  - The detail shows the vendor's **own lines only** for that order, the same stepper
    component the customer uses, the estimated delivery date, visible notes, and courier
    details when shared.
  - Existing vendor actions (`processing` / `handed_to_hb`) stay on the list.
  - Vendors cannot post tracking updates.
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
- Vendor emails for any stage other than **Delivered** (R8 covers Delivered).
- A product-aware delivery estimate (waits on SVO-3; see R9 "Later").
- Vendors posting tracking updates. Only admins drive stages beyond `handed_to_hb`.
- A public (logged-out) tracking page or tracking-number lookup.
- A real courier adapter and automatic polling.
- Backfilling timeline events for existing orders.

## Open questions (ask a human)

1. ~~Estimated delivery date fallback~~ — **resolved 2026-10-05:** route-based default
   windows held in settings (R9): domestic 7–14 days, cross-border 14–28. A
   product-aware estimate comes later, after SVO-3.
5. **SVO-3 per-product override shape** (for SVO-3's clarify step): does a product
   override hold one window, or a domestic + cross-border pair? A product ships from one
   origin to either route. Ask when SVO-3 is picked up.
2. ~~Courier tracking reference visible to customers~~ — **resolved 2026-10-05:** it is
   an admin choice per order via the sharing switch (R7).
3. **Cancelled after shipping:** the state machine forbids it today. This spec does not
   change that.
4. ~~Vendor tracking emails~~ — **resolved 2026-10-05:** Delivered only (R8, OT-6).

## Implementation Notes (OT-1, OT-2, OT-3 — 2026-10-06)

**Branch:** `feat/EUy8sgIY-order-tracking` · **PR:** pending · **Cards:** [OT-1](https://trello.com/c/EUy8sgIY) · [OT-2](https://trello.com/c/Z3gQhcTv) · [OT-3](https://trello.com/c/HD13Bk1b)

**OT-1 (API):**
- `@hb/shared` gains `TrackingStage`, `TrackingTarget`, `TrackingEventSource` enums and the `contracts/tracking.ts` suite (`OrderTrackingDto`, `AdminOrderTrackingDto`, `AddTrackingUpdateRequest`, etc.). `OrderDto.trackingStage` added; `VendorOrderLineDto.trackingStage` added for OT-5.
- Migration `1788777600000-OrderTracking`: `order_tracking_events` table with `(orderId, occurredAt)` index, `shipments` gains courier/estimate/sharing columns and a UQ index on `orderId`, platform_settings seeded with 7/14 domestic and 14/28 cross-border days.
- `OrderTrackingService` in orders module: R1 stage derivation and R4 event appending in `order-tracking.util.ts`. R3 coupled writes locked via `pessimistic_write`. Lazy shipment creation uses INSERT … ON CONFLICT DO NOTHING.
- `AdminOrderTrackingController` in orders module (to avoid cyclic imports) with `POST /admin/orders/:id/tracking` and `GET /admin/orders/:id/tracking`.
- `capturePayment`, `updateStatus`, `overrideStatus` now append tracking events inside a transaction with row-level write lock on re-read to prevent stale `deliveredAt` stamps.
- `GET /orders/:id/tracking` (owner or admin; vendor read is OT-5).
- API 1625 tests pass, lint clean.

**OT-2 (admin):**
- New **Update tracking** panel in admin order detail, rendered above the override section.
- Offers only `nextTargets` from the server; skips `nextTargets` if none available.
- Review fix: loads with notify OFF so courier changes alone don't advance the order; re-defaults `notifyByDefault` on target pick.
- Share-courier checkbox prefilled ON on first entry.
- Delivery-estimate min/max pairs added to admin settings.
- Tab added for "Handed to HB" filter; label corrected.
- Stale-response guard and double-submit lock.
- Web 1642 tests pass before review fixes; touched specs (161) re-pass after.

**OT-3 (customer):**
- New deep-linkable `/profile/orders/:id` route with R9 delivery-window branches and delivery-late reassurance copy.
- Visual stepper component (presentational, reused by OT-2) with `aria-current="step"`, UTC+2 timestamps, vertical below 640px.
- Courier block shown only when shared; Updates list newest first.
- Pending banner replaces the stepper.
- Page SCSS restored after move; copy aligned with Shipping Policy (7–14 domestic, 14–28 cross-border) and checkout/PDP.
- Full build green.

**Spec clarifications (v1 behaviour, documented):**
- **N1:** an exact `estimatedDeliveryDate` cannot be cleared once set; the DTO rejects `''` and the picker skips emptied values, so orders cannot fall back to the R9 window.
- **N2:** note-only and courier-only updates are accepted on pending, cancelled and delivered orders, emitting `TRACKING_UPDATED` when notify is on; OT-4 listener must handle cancelled targets.
- **N3:** an override from `delivered` back to `shipped` leaves the shipment at `delivered`, so the panel offers no next target and the admin must override a second time to exit `delivered`.

**Verification gap:**
- Browser check (cross-border/domestic screenshots at 375px) not yet done.
- Dev login blocked on main by SEC-5 CSP (`connect-src 'self'` vs dev `http://localhost:3000/api`); separate fix task raised; not from this batch.

**Follow-ups:**
- OT-4 (emails) and OT-5/6 next.
- Dev-server CSP/proxy fix.
- `admin-orders.scss` budget warning (2.1 kB over, down from 2.7).
