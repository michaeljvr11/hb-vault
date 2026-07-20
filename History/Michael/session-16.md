---
operator: Michael
date: 2026-07-20
session: 16
tags: [ai-factory, ship-card, storefront]
---

# Session 16 — Product listing pagination & sort

## What we worked on
Trello card **rmpsJVLi** (#44) — Product listing pagination + sort (fast-follow to Discovery API + storefront search). Shipped as pagination layer atop existing discovery/list infrastructure.

## What changed

**Shared contract (`@hb/shared`):**
- New generic envelope `PagedResponse<T> = { items: T[], total: number, page: number, limit: number }` in `libs/shared/src/contracts/common.ts`, barrel-exported.
- New `ProductSort = 'newest' | 'price_asc' | 'price_desc' | 'name'` enum.
- `ProductQuery` extended with optional `page`, `limit`, `sort` fields.

**API (`apps/api/src/products/`):**
- `GET /products` now returns `PagedResponse<ProductDto>` (breaking change; all consumers updated same PR).
- `ProductQueryDto` validates: `page`/`limit` as `@Type(() => Number) @IsInt @Min(1)` (clamp-not-reject: **no** `@Max`, limit is server-clamped); `sort` via `@IsIn(['newest', 'price_asc', 'price_desc', 'name'])`.
- `ProductsService.findAll`: `DEFAULT_LIMIT=24`, `MAX_LIMIT=100`, `DEFAULT_PAGE=1`; logic: `limit = clamp(input ?? 24, 1, 100)`, `page = max(input ?? 1, 1)`, `skip = (page-1)*limit`.
- `SORT_MAP` resolves each sort option (newest → `createdAt DESC`, price_asc/desc → `price ASC/DESC`, name → `name ASC`) with `product.id` appended as stable tiebreaker on every sort.
- **Critical invariant preserved:** ordering + skip/take appended **AFTER** approved-vendor visibility Brackets and existing categoryId/q/vendorId predicates — no sort/page/param combo can resurface non-approved vendor listings (regression-tested).
- **Default order:** newest-first (`createdAt DESC`) when `sort` omitted. Prior unordered behavior could not paginate stably; newest matches shop "New in Namibia" carousel intent.
- Uses `getManyAndCount()` for envelope `total` (atomic count, not separate query).

**Web (`apps/web/src/app/`):**
- `ProductsService.list()` returns `PagedResponse` envelope and forwards `page`/`limit`/`sort`.
- **Discovery route (`/discover`)** fully URL-driven: `page` from `?page=`, `sort` from `?sort=` (defaults 1/newest), composing with `q`/`categoryId`/`vendorId`. Sort dropdown + Prev/Next pager ("Page X of Y"). Pages replace (not accumulate); `/discover?...&page=2` SSR-renders that exact slice. Filter/sort change resets to page 1. Out-of-range `?page=` self-heals via redirect to last valid page.
- **Other list consumers** (`shop` carousel + category counts, `vendor-profile`, `vendor-products`, `admin-catalog`, `product-detail` related list) read `.items` and pass explicit `limit: 100` to avoid truncating at default 24.
- No TypeORM migration (contracts + query only).

## Key decisions

1. **Newest-first default:** required deterministic order for stable pagination (prior unordered behavior failed at page boundaries); confirmed via CLARIFY step.
2. **Generic envelope `PagedResponse<T>`:** reusable pattern for future paginated endpoints.
3. **Clamp-not-reject limit:** silently clamping user input is better UX than 400 on invalid limit.
4. **Stable tiebreaker (`product.id` on every sort):** necessary for predictable page boundaries across all sort options.
5. **Limit:100 for non-discovery lists:** adequate for shop/vendor/admin volumes in v1; real pager deferred if volumes grow.
6. **Invariant preservation:** ordering/skip/take occur AFTER visibility Brackets and predicates — regression-tested to prevent unapproved vendor leak.

## Scope & orchestration

Single card, single session. Shared + API + web (breaking API contract change handled atomically).

## Test/review outcome

- **API:** 435/435 tests pass (new suite: pagination math, sort map resolution, clamp/max behavior, invariant that unapproved vendors stay invisible across all sort/page combos, DEFAULT_LIMIT/MAX_LIMIT).
- **Web:** 613/613 tests pass (discover pagination UI, sort dropdown, Prev/Next pager, out-of-range redirect, page reset on filter change, list consumers reading `.items`).
- **Lint:** `npm run lint:api` → clean.
- **Build:** `npm run build` (shared→api→web) → passes.
- **Code-reviewer:** SHIP. One WARN (out-of-range page dead-end) was fixed (self-heal redirect now applied).

## PR status

PR #36 — https://github.com/michaeljvr11/hb-mono-repo/pull/36 — open, awaiting human merge.

Branch: `feat/rmpsJVLi-product-pagination-sort`.

## Related notes

[[Category Taxonomy & Discovery]] — Implementation Notes section added; "Out of scope" table updated to reflect pagination/sort SHIPPED status.

## Follow-ups

- Shop's client-side category counts degrade above 100 products (candidate for server-side aggregate-count endpoint or real pager).
- Case-insensitive `name` sort (`LOWER(name)`) deferred — risks DISTINCT-pagination SQL; revisit if UX pressure mounts.
- Minor constant-consolidation nits (e.g., `PRODUCT_LIST_MAX` as named constant in shared schema).
