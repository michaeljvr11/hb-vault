---
operator: Michael
date: 2026-10-06
session: 54
tags: [ai-factory, ship-batch, order-tracking]
---

# Session 54 — OT-4/5/6 order-tracking batch

**Date:** 2026-10-06 · **Cards:** [OT-4](https://trello.com/c/zSqz4wBg) (customer emails) · [OT-5](https://trello.com/c/25OQwbvD) (vendor tracker) · [OT-6](https://trello.com/c/qJrrBvaJ) (vendor delivered email) · **Branch:** `feat/zSqz4wBg-order-tracking-notifications` · **Status:** PR #117 open

## Shipped

**OT-4:** `OrderTrackingNotificationsListener` on `OrderEvents.TRACKING_UPDATED` (best-effort safe), milestone-keyed email subjects (shipped/border/delivery/delivered; generic fallback), review fixes for email-kind derivation from target (not snapshot diff) and estimate gating on DTO stage. New `OrderTrackingService.getCustomerTracking()` centralizes customer-safe logic.

**OT-5:** Vendor read access to tracking (`GET /orders/:id/tracking` with `vendorHasLine` check), `VendorOrderLineDto.trackingStage` in shared contracts (no N+1 shipment query). Web: stage badge on list, deep-linkable `/vendor/orders/:orderId` detail (vendor's own lines only, read-only). Fixed responsive overflow at 375px with `flex-wrap`.

**OT-6:** `OrderEvents.DELIVERED` on first-delivery only (conditional `UPDATE affected === 1` check), `OrderDeliveredNotificationsListener` with per-vendor email isolation (own lines, no commission/customs). Payout window from `DAMAGE_CLAIM_WINDOW_HOURS`, link `/vendor/orders/<id>`.

## Tests

API 1717 tests (100 suites), Web 1661 tests (108 files), lint clean, build clean. Code-reviewer fix-first on 3 OT-4 items (all fixed); authorization/links/data-leak checks passed.

## Follow-ups

Shared tracking-panel component (customer + vendor pages candidate cleanup).
