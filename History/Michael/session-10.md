---
operator: Michael
date: 2026-06-29
session: 10
tags: [ai-factory, ship-card, products, public-api]
---

# Session 10 — Product filter by approved vendor status

## What we worked on
Trello card **c2o6xfZs** (#36) — "Filter vendor listings by approved status on public product endpoints". API only. This card extends the approved-only visibility rule already shipped for the vendor directory and vendor profiles to the public product catalogue endpoints.

## What changed

**API only — no `@hb/shared`, no migration, no frontend:**
- `apps/api/src/products/products.service.ts` — `findAll` and `findOne` now filter public product results via FindOptions `where` array-of-conditions (OR-of-AND): return products where `listingType = 'platform'` OR (`listingType = 'vendor'` AND `vendor.status = 'approved'`). Platform (first-party) listings always visible. Non-approved vendors' products are absent from `GET /products` and 404 on `GET /products/:id` (hits existing `NotFoundException`) — even by direct id. Imports `VendorStatus` from `@hb/shared` (already exported).
- `apps/api/src/products/products.service.spec.ts` — NEW, 14 unit tests: `findAll` (platform always visible, approved vendors visible, pending/rejected/suspended excluded via real WHERE-array mock, exact where/relations assertion, empty repo, mixed fixture); `findOne` (platform + approved resolve, each non-approved status → 404 by direct id, unknown id → 404).

## Key decisions

**Applied the same approved-only rule already shipped for vendors:**
- Card vs3y5tDJ / PR #21 shipped `GET /vendors/directory` and `GET /vendors/:id` with approved-only filtering. This card replicates that pattern on the product catalogue for consistency and to prevent non-approved vendors' products from leaking via discovery or direct lookup.

**Service/query-layer filter, no controller change:**
- Reuses the existing visibility pattern. No new endpoint created. The change is transparent to the controller.

**Chose FindOptions array-of-conditions over QueryBuilder:**
- Minimal diff; the `vendor` relation is ManyToOne so the nested-relation join is safe and does not truncate the eager `images`/ManyToMany `categories` collections.

**No `@hb/shared` change, no migration, no schema change:**
- All fields already exist. This is a visibility + filtering change only.

## Test & review outcome

- **API tests:** `npm run test:api` → **126 passed (+14 new in products.service.spec.ts)**.
- **Web tests:** `npm run test -w @hb/web` → 293 passed (no web files touched).
- **Build:** `npm run build` → clean (only pre-existing `shop.scss`/`admin-catalog.scss` budget warnings, unrelated).
- **Lint:** `npm run lint:api` → clean.
- **Code review:** code-reviewer (opus) — SHIP. No FAILs. One forward-looking follow-up (out of scope for #36, relevant to card #37 "Category filter + search on GET /products"): if `findAll` ever gains pagination, this FindOptions shape with collection joins would truncate `images`/`categories` and must move to a QueryBuilder with `relationLoadStrategy: 'query'` or distinct pagination. Card #37 also edits `findAll` and must AND its predicates with this approved-vendor filter so no non-approved listing leaks via any param.

## PR status
PR **#23** (https://github.com/michaeljvr11/hb-mono-repo/pull/23) — open, awaiting human merge. NEVER merged by AI; a human owns prod.

## Follow-ups

- **Card #37 composition:** "Category filter + search on GET /products" edits `findAll` and must AND its predicates with this approved-vendor filter. No non-approved listing must leak via any param combination.
- **Pagination on `findAll`:** if products pagination is added, this FindOptions shape must migrate to QueryBuilder with `relationLoadStrategy: 'query'` or distinct pagination to avoid collection truncation.

## Orchestration

- **Single card, single layer:** API only, no `@hb/shared`/migration/frontend. Service/spec straightforward composition.
- **Roles:** backend-engineer (implemented), test-engineer (spec coverage), code-reviewer (opus, SHIP).
- **No Agent Team:** tight scope, single delivery vehicle.
- **Docs slice** (this session): `Listing Types & Vendor Rules.md` Implementation Notes section + this session log + evidence headline refresh.
