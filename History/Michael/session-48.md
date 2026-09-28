# Session 48 — 8QGKsOPB, e1axOfc1, qWOUiFr5, roDUKEED, 3nInXOrB: Taxonomy, Theme, Cart, Vendor Details Batch

**Date:** 2026-09-28 · **Cards:** 8QGKsOPB, e1axOfc1, qWOUiFr5, roDUKEED, 3nInXOrB · **Branch:** feat/8QGKsOPB-taxonomy-theme-vendor-cart-batch · **Status:** PR pending

- **Shipped:** Two-level category taxonomy (parentId selector, cycle guard, depth cap 2, idempotent seed), light/dark/system theme toggle in settings + nav + radial menu (signal + localStorage, pre-paint script for FOUC), cart vendor grouping (CartItemDto.vendor, groups inside currency), vendor productCount on public directory (approved-only), vendor application fields (businessDescription, isRegisteredBusiness, registrationNumber, productCategoryIds, contactName, contactPhone, migration).
- **Decisions:** Depth=2 cap (no n-level recursion); category reparent checked via recursive walk (admin-only, not transactional); theme system=no-class; vendor count omitted from admin/self; contactPhone admin-only.
- **Tests:** api 1268/1268, web 1527/1527, lint clean, build green (requires INTERNAL_API_BASE_URL for SSR).
- **Follow-ups:** DTO parentId typing (string|null), nav-bar SCSS budget, phone validation duplication (web), registrationNumber empty-string coercion.
