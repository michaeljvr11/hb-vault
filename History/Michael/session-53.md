---
operator: Michael
date: 2026-10-06
session: 53
tags: [ai-factory, ship-batch, order-tracking]
---

# Session 53 — Order tracking batch (OT-1/2/3)

**Date:** 2026-10-06 · **Cards:** [OT-1](https://trello.com/c/EUy8sgIY) (API foundation) · [OT-2](https://trello.com/c/Z3gQhcTv) (admin panel) · [OT-3](https://trello.com/c/HD13Bk1b) (customer tracker) · **Branch:** `feat/EUy8sgIY-order-tracking` · **Status:** PR to be opened

## Shipped

**OT-1:** API tracking timeline (`order_tracking_events` table, R1–R4 rules), lazy shipment with UQ index, platform-settings seeded 7/14 domestic and 14/28 cross-border days, `OrderTrackingService` in orders module with coupled transactional writes and pessimistic lock on status/override, `AdminOrderTrackingController` for admin update/read endpoints, `@hb/shared` TrackingStage/Target/EventSource enums and full DTO suite (OrderTrackingDto, AdminOrderTrackingDto, AddTrackingUpdateRequest). Verified: migration up/down/up on dev, API 1625 pass, lint clean.

**OT-2:** Update tracking panel in admin orders detail, targets select re-defaults notify from server, share-courier checkbox prefilled ON, delivery-estimate settings card added, "Handed to HB" tab + label fix, stale-response guard and double-submit lock. Web 1642 pass before review fixes; 161 touched specs re-pass after.

**OT-3:** Deep-linkable `/profile/orders/:id` route with R9 delivery-window branches and late-running copy, shared stepper component (aria-current, UTC+2, vertical <640px), courier block on share, Updates list newest-first, pending banner, SCSS restored, copy aligned Shipping Policy + checkout/PDP. Full build green.

## Tests

- API 1625 pass, lint clean.
- Web 1642 pass (review fixes on 161 specs).
- Build clean (needs `INTERNAL_API_BASE_URL` = CI placeholder locally).

## Follow-ups

- OT-4 (emails), OT-5 (vendor read), OT-6 (vendor delivered email).
- Dev-server CSP/proxy fix (SEC-5 blocker, pre-existing).
- Browser 375px check (cross-border/domestic screenshots).
- `admin-orders.scss` 2.1 kB budget overage.
