# Session 11 — PD-1..PD-5: Product Discounts & Sale Pricing batch
**Date:** 2026-09-07 · **Card:** PD-1 UdeL4byD, PD-2 WY3NxxLs, PD-3 AchyX7xS, PD-4 1ceMr9wa, PD-5 J7NivPZU · **Branch:** `feat/UdeL4byD-product-discounts` · **Status:** open (PR not yet opened)

- Shipped: server-resolved discount percent + scheduled start/end window on products (`isDiscountActive`/`effectivePrice` always present on `ProductDto`), snapshotted onto order lines at creation, vendor + admin UI to set it, storefront struck-price/badge to show it. Also fixed the cart reading live price instead of the discounted one (owner-approved, beyond the five cards).
- Decisions: discount cap 90% is a fat-finger guard not a business ceiling; admin platform listings get discounts in v1; `PATCH /products/:id` omitting `discountPercent` clears the discount — both web callers always send all three fields to avoid wiping it accidentally.
- Tests: api 75/75 suites (1032 tests), web 85 files (1277 tests), lint clean, build clean, migration verified up/down/up on PG16, code-reviewer SHIP with zero FAILs.
- Follow-ups: `docs/design/DESIGN.md`'s "API has neither field" caveat on the sale-badge slot is now stale, needs a touch-up next time that doc is edited.
