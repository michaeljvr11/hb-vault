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

## Implementation Notes — Discovery API + storefront search UI (slices 1+2, cards b4VoyjRu + 6qlkwk75)

**2026-07-02 — SHIPPED (PR #26)**

**What shipped (single branch, owner-approved dual-card exception):**
- **API discovery filters** — `GET /products` now accepts optional `?categoryId=`, `?q=`, `?vendorId=` query params validated by `ProductQueryDto` (implements `ProductQuery` in `@hb/shared`). Category filter matches products in that category; search is case-insensitive over product `name` + `description`; vendor filter narrows to a specific vendor's products. All compose together and with the approved-vendor-only visibility filter (via TypeORM Brackets).
- **Unified suggest endpoint** — `GET /search/suggest?q=` (new module `apps/api/src/search/`) groups suggestions into `Products`, `Vendors`, `Categories` with product images/stats, vendor names/logos, category names. Populated from both approved vendors and platform listings.
- **Storefront browse + search UI** — `/shop` rebuilt to desktop + NEW mobile designs; mobile sticky search + category chips deep-link to `/discover`. Category grid filters live on click. New omnibox search-bar with grouped suggestions + keyboard nav (arrow/Enter/Escape) + ARIA labels. Dismissible vendor chip on results.
- **Shared component foundation** — product-card grid/carousel, category-chips, search-bar omnibox, trust-banner, vendor-showcase, radial-nav (Angular port of mobile design). 21 files, Material + design-token adherence.
- **Product detail route** — new SSR route `/products/:id` with gallery, shipping card, vendor card (View Storefront→/discover?vendorId=), related products, sticky add-to-cart + anon→login returnUrl gate, 404 state. Single-currency price display (ProductDto has one price+currency).
- **Discovery browse route** — new SSR route `/discover` with URL-driven state (q/categoryId/vendorId query params), SME client-side toggle (filters where product.vendor presence), distinct empty state for each filter combo.

**Scope expansion (owner-approved):**
- `vendorId` param + unified suggest endpoint shipped beyond spec v1. Owner requirement: "vendor-specific marketplace → users search vendors/categories/products". Vendor suggestions land on `/discover?vendorId=` (no vendor-profile page/design exists yet; created follow-up card #40 UG5UFyxy).
- Product detail is single-currency (ProductDto has one price+currency). Design's dual ZAR/NAD parity display deferred — no peg source client-side. Future card for currency/locale handling.

**Review finding — precedence bug + false-green regression:**
- **THE BUG:** unparenthesized OR in SQL visibility clause let platform listings bypass ALL category/vendor/search filters (OR binds weaker than AND). Wrapped in TypeORM Brackets; all filters now properly AND together.
- **FALSE-GREEN SPECS:** `ProductsService.findAll` tests used fake QueryBuilder lacking SQL AND/OR semantics, so the precedence bug was invisible. Added regression guards that fail on pre-fix code + precedence-faithful fake QueryBuilder.
- **Lesson:** even with full mock coverage, semantic testing of query composition needs fake implementations that respect SQL operator precedence. The guardrails caught what unit test coverage missed.

**Decisions (confirmed via review + owner):**
- All filters compose via AND in a single query builder (no separate queries or client-side combining).
- Approved-vendor visibility applies; vendors pending/rejected/suspended stay off discover.
- Client-side SME toggle is UX-only (filters by product.vendor presence); no API gate.
- Pagination/sort stays deferred (fast-follow card #44 rmpsJVLi created).

**Contract/schema:** 
- New `@hb/shared` `ProductQuery` interface + suggest response types.
- No TypeORM schema change (all columns already exist).

**Tests:**
- `apps/api/src/products/products.service.spec.ts` — 12 new suites covering each filter in isolation + composition + approved-vendor AND-ing. False-green fake QueryBuilder replaced.
- `apps/web/src/app/pages/discover/` + `products/` — 37 Vitest specs (discover state, routing, SME toggle, empty states, product-detail gallery/vendor-card/add-to-cart, anon→login return).

**Test/build outcome:**
- `npm run test:api` → 175 passed (12 suites).
- `npm run test -w @hb/web` → 395 passed (37 files).
- `npm run lint:api` → clean.
- `npm run build` → clean.
- **Code review:** FIX-FIRST (2 FAILs: precedence bug + false-green specs; 2 WARNs: SME badge color + param source). All addressed.

**PR:** #26 (https://github.com/michaeljvr11/hb-mono-repo/pull/26) — open, awaiting human merge.

**Follow-ups:**
- Card #40 UG5UFyxy (vendor profile page with stats/reviews — vendor suggestions currently land on /discover?vendorId=).
- Card #44 rmpsJVLi (pagination + sort — results unbounded in v1).
- DesignSync push to claude.ai (needs interactive `/design-login`; deferred to Josh).

---

## Vertical slices → Trello cards
| # | Title | Layer | Card | Status |
|---|---|---|---|---|
| 1 | Category filter + search on `GET /products` | api + shared | #37 `b4VoyjRu` | SHIPPED (PR #26, fable-storefront-oneshot) |
| 2 | Storefront: browse-by-category + search UI | web | #38 `6qlkwk75` | SHIPPED (PR #26, fable-storefront-oneshot) |
| 3 | Safe category delete (block in-use / parent) | api | #39 `Sm1kSNO8` | SHIPPED (PR #24) |
| 4 | Pagination + sort on discovery / list views | shared + api + web | #44 `rmpsJVLi` | SHIPPED (PR #36) |

Recommended order: 1 → 2 (2 depends on 1). Slice 3 is independent and can run in parallel.
Sequence slice 1 with card `c2o6xfZs` (#36) so the discovery filter and the approved-vendor
filter compose in one query builder. Slice 4 is a fast-follow to slices 1–2.


---

## Implementation Notes — Pagination + sort (fast-follow, card #44 rmpsJVLi)

**2026-07-20 — SHIPPED (PR #36)**

**Contract (`@hb/shared`):**
- New generic envelope `PagedResponse<T> = { items: T[], total: number, page: number, limit: number }` in `libs/shared/src/contracts/common.ts`, barrel-exported.
- New `ProductSort = 'newest' | 'price_asc' | 'price_desc' | 'name'` enum.
- `ProductQuery` extended with optional `page`, `limit`, `sort` fields.

**API (`GET /products` now paginated & sortable):**
- Returns `PagedResponse<ProductDto>` (breaking change — all consumers updated same PR).
- `ProductQueryDto` validates: `page`/`limit` as `@Type(() => Number) @IsInt @Min(1)` (deliberately **no** `@Max` — limit is server-clamped, not rejected); `sort` via `@IsIn(['newest', 'price_asc', 'price_desc', 'name'])`.
- `ProductsService.findAll`: constants `DEFAULT_LIMIT=24`, `MAX_LIMIT=100`, `DEFAULT_PAGE=1`; logic: `limit = clamp(limit ?? 24, 1, 100)`, `page = max(page ?? 1, 1)`, `skip = (page-1)*limit`.
- **Sort map:** `SORT_MAP` resolves each sort option to a primary column+direction, with `product.id` appended as a stable tiebreaker on every sort (preserves page integrity).
- **Critical invariant:** ordering + skip/take appended **AFTER** the approved-vendor visibility Brackets clause and existing `categoryId`/`q`/`vendorId` predicates — no sort/page/filter combo can resurface unapproved vendor listings (regression-tested).
- **Default order:** `newest` (createdAt DESC, id tiebreak) when `sort` omitted. Prior unordered behavior could not paginate stably; newest matches the shop "New in Namibia" carousel intent (confirmed via CLARIFY step).
- Uses `getManyAndCount()` for envelope `total` (not separate count query).

**Web (`ProductsService.list()` + consumers):**
- `ProductsService.list()` returns the `PagedResponse` envelope and forwards `page`/`limit`/`sort`.
- **Discovery route (`/discover`)** is fully URL-driven: `page` from `?page=`, `sort` from `?sort=` (parsed/validated, defaults 1/newest), composing with existing `q`/`categoryId`/`vendorId`. Features sort dropdown + Prev/Next pager ("Page X of Y"); pages **replace** (not accumulate) so `/discover?...&page=2` SSR-renders that exact slice. Changing any filter or sort resets to page 1. Out-of-range `?page=` self-heals via redirect to last valid page.
- **Other list consumers** (`shop` carousel + category counts, `vendor-profile`, `vendor-products`, `admin-catalog`, `product-detail` related list) read `.items` and pass explicit `limit: 100` (`PRODUCT_LIST_MAX`) to avoid truncating at default 24.
- No migration (contracts + query only).

**Tests/review:**
- `npm run test:api` → 435 passed (new suite: pagination math, sort map resolution, invariant: unapproved vendors remain invisible across all sort/page combos, DEFAULT_LIMIT/MAX_LIMIT behavior).
- `npm run test -w @hb/web` → 613 passed (new discover pagination UI, sort dropdown, Prev/Next pager, out-of-range redirect, page reset on filter change, list consumers reading `.items`).
- `npm run lint:api` → clean.
- `npm run build` (shared→api→web) → passes.
- **Code review:** SHIP; one WARN (out-of-range page dead-end) was fixed (self-heal redirect).

**Decisions (from CLARIFY confirmation):**
- Default order = **newest-first** (createdAt DESC) when `sort` omitted — required deterministic default to avoid page-0 instability.
- Envelope generic `PagedResponse<T>` reusable across future paginated endpoints.
- Clamp-not-reject limit (400 on invalid limit is worse UX than silently clamping).
- Stable tiebreaker (`product.id` appended to every sort) — necessary for predictable page boundaries.
- List consumers cap at `limit: 100` (fine below 100 products; candidate for real pager if shop/vendor/admin-catalog exceed that volume).

**Follow-ups (deferred, bounded):**
- Shop's client-side category counts degrade above 100 products (candidate for server-side aggregate-count endpoint or real pager).
- Case-insensitive `name` sort (`LOWER(name)`) deferred — risks the `getManyAndCount` DISTINCT-pagination SQL; can revisit if UX pressure mounts.
- Minor constant-consolidation nits (e.g., `PRODUCT_LIST_MAX` as a named constant in schema/contracts).

**PR:** #36 (https://github.com/michaeljvr11/hb-mono-repo/pull/36) — open, awaiting human merge.
