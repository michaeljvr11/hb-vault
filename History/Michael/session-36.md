# Session 36 — 23PYFGW7/7XSMAVih: Product reviews edit/delete (final batch)

**Date:** 2026-08-23 · **Cards:** [23PYFGW7](https://trello.com/c/23PYFGW7) / [7XSMAVih](https://trello.com/c/7XSMAVih) · **Branch:** `feat/23PYFGW7-edit-delete-own-review` · **Status:** Open

- Shipped: PR-5 API author-only edit/delete (`PATCH /reviews/:id`, `DELETE /reviews/:id` 204, flat separate controller from product-scoped Reviews controller, ownership enforced by service.findOwnedReview {id, userId} → 404 if non-owned, immutability via selective field spread + whitelist guard). PR-6 web edit/delete UI on PDP (edit reuses PR-4 form signals pre-filled; delete inline confirm; both re-fetch after success, no local splice; error messaging hoisted to sibling of eligibility branches to prevent unmount-on-refetch, same fix pattern as PR-4 success confirmation).
- Decisions: Flat controller `/reviews/:id` per spec (not nested under `/products/:productId/reviews/:id`), mirroring established ownership-check pattern. Edit/delete both trigger full refetch (never local state mutation). Empty {} PATCH body explicitly 400 rejected.
- Tests: api 860/860 · web 930/930 · lint clean · build clean (SCSS budget warning noted, out of scope).
- Follow-ups: SCSS `product-detail.scss` 7.24kB over budget (markup duplication from failed ngTemplateOutlet attempt); low priority.
