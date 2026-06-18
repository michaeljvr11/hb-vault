---
operator: Josh
date: 2026-06-18
session: 2
tags: [ai-factory, ship-card, storefront]
---

# Session 2 — Storefront (L9fM1Gu7)

## What we worked on
Card L9fM1Gu7 "Implement Store front" — storefront page rendered after login at `/shop`, displaying Products, Categories, and Vendors with nav bar and footer.

## What changed

**Backend:**
- New PUBLIC `GET /vendors/directory` endpoint returning only `approved` vendors via `VendorsService.findDirectory()` with unit test. Existing admin-only `GET /vendors` unchanged.
- Idempotent seed script `apps/api/src/database/seed.ts` + `npm run seed` — seeds 4 categories, ~10 products (platform + vendor mixed), and 4 approved SME vendors (no user accounts; `userId` null). Honors `vendorId`/listing-type invariant + DB CHECK. No schema change.

**Frontend:**
- Standalone `NavBar` + `Footer` components under `apps/web/src/app/layout/`.
- `/shop` page: hero → "New in Namibia" product carousel → trust banner → "Explore Categories" grid → "Featured SME Vendors" → newsletter → footer.
- Idiomatic Angular (standalone + signals + new control flow) with scoped SCSS over `--hb-*` tokens (no Tailwind in-app).
- Wired to Products/Categories/Vendors services; signals drive per-section loading/empty/error.
- Missing-image products get branded placeholder; "SME Verified" badge from `status === 'approved'`.
- Prices format ZAR→`R`, NAD→`N$`.
- Pure `deriveCategoryCounts()` helper (unit-tested) derives category product-counts client-side.

## Key decisions

- Vendor directory is public but returns approved-only (business rule already in "Listing Types & Vendor Rules").
- Honest-to-data design: `VendorDto` has no rating/city/logo and `CategoryDto` has no image, so visual elements from Claude Design export are omitted or derived (e.g. counts) rather than faked.
- Cart, search, and ZAR/NAD currency switcher are presentational stubs (no backing feature yet).

## Test & review outcome

API tests 25 ✓ · Web tests 50 ✓ · `lint:api` clean · `npm run build` passes (one non-blocking `shop.scss` style-budget warning). Code-reviewer verdict **SHIP** (no blocking issues). NITs addressed: extracted + tested `deriveCategoryCounts`, removed a dead signal.

## Follow-ups

- Reconcile hero accent colors to DESIGN.md tokens (or add sanctioned tokens).
- Consider lifting shared style primitives (`.btn`, `.state-message`, `.section`) into a shared stylesheet to drop storefront style budget.
- Seed idempotency is coarse (all-or-nothing on category count); refactor per-entity later.
- Cart/search/currency-switcher await their own cards.
- If storefront later needs vendor logos/ratings/city, that's a `Vendor` model + contract change.

**Branch:** `feat/L9fM1Gu7-storefront` (4 commits, 2026-06-11 → 2026-06-18). **PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/4
