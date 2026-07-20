---
operator: Michael
date: 2026-07-20
session: 15
tags: [ai-factory, ship-card, storefront]
---

# Session 15 — Public vendor profile page

## What we worked on
Trello card **UG5UFyxy** (#40) — Public vendor profile page (`/vendors/:id`) — design + web UI. Shipped as slice 5 of the Public Storefront & SSR epic.

## What changed

**Frontend (`apps/web/src/app/features/vendors/vendor-profile/`):**
- New `PublicVendorProfile` component (signals-based, OnPush, SSR-safe). Fetches vendor by `:id` param via `VendorsService.getById` (public endpoint from slice 3 / PR #21, approved-only) and listings via `ProductsService.list({ vendorId })`.
- Hero section: business name, optional trading name, "Verified SME" badge (all public vendors are approved), country label from `countryCode`. Category-tag chips derived client-side (distinct product categories, de-duped by id).
- Product grid reuses shared `ProductCard`. Anonymous add-to-cart → `/login?returnUrl=…` (established slice-2 boundary). Error/loading/empty/not-found states handled.
- Route `/vendors/:id` is public (no guard), placed before `**` catch-all in `app.routes.ts`, and `RenderMode.Server` in `app.routes.server.ts`.
- Re-pointed three "visit this vendor" affordances: product-detail `viewStorefront()`, discover omnibox "Vendors" suggestion, shop-home vendor-showcase tiles (`shop.onVendorSelect`). Discover `?vendorId=` filter machinery left intact.
- Design artifact (no Claude Design source, login unavailable): `docs/design/claude-design/screens/vendor-profile/index.html` + mirror export.

**No API / @hb/shared change.**

## Key decisions

1. **Component name `PublicVendorProfile`** — avoids duplicate class name with vendor-portal `VendorProfile`.
2. **Client-side category chips** — distinct product categories de-duped by id; no API expansion (keeps contract lean).
3. **All three affordances re-point** — clarity with dev: discover `?vendorId=` filter left working so deep-linked URLs still resolve.

## Scope & orchestration

Single card, single session. Web + design only.

## Test/review outcome

- **Web:** 600/600 tests pass (incl. new vendor-profile.spec.ts: happy path, products by vendorId, 404 not-found, missing id, generic error, empty state, category de-dupe, country-label mapping, anonymous add-to-cart gate; updated product-detail/discover/shop re-point specs).
- **API:** 406/406 tests pass (untouched).
- **Lint:** clean.
- **Build:** full SSR build clean (only pre-existing `admin-catalog.scss` budget warning).
- **Code-reviewer:** SHIP, 0 FAIL / 2 NIT (applied both: rename to `PublicVendorProfile`; SME-badge border via `color-mix` over `--hb-primary` instead of hardcoded rgba). Left `takeUntilDestroyed` NIT to match product-detail sibling pattern.

## PR status

PR #35 — https://github.com/michaeljvr11/hb-mono-repo/pull/35 — open, awaiting human merge.

## Related notes

[[Public Storefront & SSR]] — slice 5 now documented; cross-note updated (vendor-profile done in PR #35).

## Follow-ups

- `docs/design/vendor-profile/reference.png` screenshot not captured (Docker Desktop offline; optional for docs refresh).
- Optional: list `vendor-profile` as implemented in `apps/web/CLAUDE.md` + `DESIGN.md` screens table.
