# Category Taxonomy & Discovery

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Listing Types & Vendor Rules]] · [[Public Storefront & SSR]] · [[Vendor & Admin Portals]].

## Problem

Categories are the product taxonomy that ties the three surfaces together: admins curate
them, vendors tag their listings with them, and customers use them to find products. Two of
those three legs already work — the third (customer discovery) does not, and the admin leg
has a data-integrity hole.

**What already exists (do NOT rebuild):**

- **Admin management** — full CRUD in `apps/api/src/categories/` (admin-only
  `POST`/`PATCH`/`DELETE`, public `GET`/`GET :id`), the `@hb/shared` contract
  (`CategoryDto`, `CreateCategoryRequest`, `UpdateCategoryRequest`), and the admin UI
  (card AP-8 / `IhTVdNYP`, `admin-catalog`). Supports `name`, `slug`, `description`,
  `displayOrder`, and `parentId` self-referencing subcategories.
- **Vendor onboarding** — `vendor-products` loads categories and has a working multi-select
  picker (`toggleCategory`); `categoryIds` flow on create/edit and are validated in
  `ProductsService.create` (invalid id → 400).
- **Storefront display** — `shop` renders a category grid with client-derived product
  counts (`deriveCategoryCounts`).

**The gaps this spec closes:**

1. **The storefront category grid is decorative.** Clicking a category does not filter the
   product list. `GET /products` takes no category param and returns everything.
2. **There is no search.** No `q` param on `GET /products`, no search box anywhere.
3. **Admin category delete is unguarded.** `CategoriesService.remove` hard-deletes via
   `categoryRepository.delete(id)` with no check for products in the category or child
   subcategories — deleting a parent orphans its children (dangling `parentId`) and silently
   strips the product↔category links.

Goal: customers can browse by category and search the catalogue (server-driven, SSR-safe),
and admins cannot silently corrupt the taxonomy by deleting an in-use or parent category.

## Scope

1. **API product discovery** — `GET /products` accepts optional `?categoryId=` and `?q=`
   query params (validated DTO implementing a new `@hb/shared` interface). Category filter
   matches products linked to that category; search is case-insensitive over product
   `name` + `description`. Both compose with each other.
2. **Storefront browse + search UI** — the category grid becomes a live filter; add a
   search input. Active-filter + clear-filter state, empty states, SSR-safe.
3. **Safe category delete** — `CategoriesService.remove` refuses to delete a category that
   has products or child subcategories (HTTP 409 + clear message); the admin UI surfaces
   the conflict.

## Business rules it must honour

From [[Listing Types & Vendor Rules]], [[Public Storefront & SSR]] — enforce, do not reinvent:

- **Browse is public.** `GET /products` is `@Public()` and SSR-rendered (`/shop` is
  `RenderMode.Server`). New query params must work without a token and on the server.
- **Approved-vendor visibility composes with discovery.** Card `c2o6xfZs` (#36) adds an
  approved-only vendor filter to `ProductsService.findAll`/`findOne`. The category/search
  filter must AND with it in the same query — a category filter must never resurface a
  pending/rejected/suspended vendor's listing. **Sequencing:** land #36 first, or build both
  in one query builder; do not let one overwrite the other.
- **Categories are a shared contract.** Filtering keys off existing `CategoryDto.id`
  (and/or `slug`); no per-surface category model. Never hand-duplicate the contract.
- **SSR safety is non-negotiable.** No `window`/`localStorage`/`document` outside
  `isPlatformBrowser`. Filter state derives from route/query, not browser globals.
- **Taxonomy integrity.** A category referenced by products or acting as a parent is in use;
  deletion must not orphan children or strip product links.

## @hb/shared contract impact

- **New:** `ProductQuery` interface (e.g. `{ categoryId?: string; q?: string }`) in
  `libs/shared/src/contracts/product.ts`, implemented by an API `ProductQueryDto`
  (class-validator). Reuses existing `CategoryDto`/`ProductDto`.
- No change to `CategoryDto` or the category create/update requests.

## Out of scope

- Pagination / sort / price-range facets — discovery v1 is single-category + text search.
  Pagination is a fast follow once result sets grow.
- Multi-category (AND/OR) filtering — single active category in v1.
- Search relevance ranking / full-text index — simple `ILIKE` over name+description in v1;
  Postgres FTS or a search service is a later decision if volume demands it.
- Category reassignment/merge tooling on delete — v1 blocks delete with 409; bulk reparent
  or "move products to…" is a future admin card.
- Re-speccing admin CRUD or vendor category-picking — already shipped.

## Open questions (confirm before /ship-card)

1. **Filter key — `categoryId` or `slug`?** Recommendation: `categoryId` for the API
   (stable, already on `CategoryDto`); slug can be a later SEO-friendly alias.
2. **Delete on in-use category — block vs. reassign?** Recommendation: block with 409 in v1
   (smallest safe change); add reassign/merge later.
3. **Parent delete — block vs. cascade-to-children?** Recommendation: block while children
   exist (forces explicit intent); revisit if admins want cascade.
4. **Search fields — name+description, or name only?** Recommendation: name + description.

## Implementation Notes — Safe category delete (slice 3, card Sm1kSNO8)

**2026-06-30 — SHIPPED**

**API guard (`CategoriesService.remove`):**
- Now loads category with `products` and `children` relations before deletion.
- Refuses deletion with HTTP 409 `ConflictException` if either is non-empty:
  - Products: "Cannot delete a category that has products; reassign them first"
  - Children: "Cannot delete a category with subcategories; delete or reparent them first"
- Unknown id still returns 404 (via `findOne` null-check, replacing old `delete().affected===0` path).
- Only an empty category is deleted via `repository.remove(category)`.
- Location: `apps/api/src/categories/categories.service.ts`

**Admin UI error surfacing (`admin-catalog`):**
- `deleteCategory` captures `HttpErrorResponse` and sets `categoryDeleteError` signal (`err.error?.message` with generic fallback).
- 409 message rendered as `error-banner` in categories section.
- Error clears when a new delete is confirmed.
- Category row is preserved on 409 conflict.
- Location: `apps/web/src/app/features/admin/pages/admin-catalog/`

**Decisions (from spec, confirmed):**
- v1 **blocks** with 409 — no auto-reassign or cascade-to-children.
- Bulk reparent / "move products to…" remains a future card.

**Contract/schema:** No `@hb/shared` change, no migration (swap from `delete(id)` to `findOne` + `remove`).

**Tests:**
- `apps/api/src/categories/categories.service.spec.ts`: 4 new cases (empty delete, 404 unknown id, 409 blocked-by-products, 409 blocked-by-children).
- `apps/web/src/app/features/admin/pages/admin-catalog/admin-catalog.spec.ts`: 2 new specs (409 error surfacing, stale-error clear on new confirm).

**Test/build outcome:**
- `npm run test:api` → 130 passed.
- `npm run test -w @hb/web` → 295 passed.
- `npm run lint:api` → clean.
- `npm run build` → clean.
- **Code review:** SHIP. Forward-looking follow-up noted: `remove` hydrates full `products`/`children` collections to check emptiness; could switch to count/exists checks if category product volume grows.

**PR:** #24 (https://github.com/michaeljvr11/hb-mono-repo/pull/24) — open, awaiting human merge.

**Follow-ups:**
- Count-based emptiness check (query optimization for high-volume categories).
- Storefront / admin reassign tooling is a separate future card.

---

## Vertical slices → Trello cards
| # | Title | Layer | Card |
|---|---|---|---|
| 1 | Category filter + search on `GET /products` | api + shared | #37 `b4VoyjRu` |
| 2 | Storefront: browse-by-category + search UI | web | #38 `6qlkwk75` |
| 3 | Safe category delete (block in-use / parent) | api | #39 `Sm1kSNO8` |

Recommended order: 1 → 2 (2 depends on 1). Slice 3 is independent and can run in parallel.
Sequence slice 1 with card `c2o6xfZs` (#36) so the discovery filter and the approved-vendor
filter compose in one query builder.
