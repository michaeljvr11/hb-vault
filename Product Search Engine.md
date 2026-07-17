# Product Search Engine

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Category Taxonomy & Discovery]] · [[Listing Types & Vendor Rules]] · [[Public Storefront & SSR]] · [[Money & Currency Rules]].

## Problem

Discovery today is a single `ILIKE '%q%'` over product `name` + `description` on
`GET /products` (card #37 `b4VoyjRu`, see [[Category Taxonomy & Discovery]]). That note
**explicitly deferred** real relevance ranking / full-text indexing "if volume demands it".
This spec is the resolution of that deferral: a dedicated marketplace search engine with
typo tolerance, field weighting, business-rule-aware ranking, faceted filters, autocomplete,
and domain synonyms.

`ILIKE` gives no typo tolerance, no relevance ranking (a title match ranks equal to a
description match), no facets, and degrades linearly as the catalogue grows.

## Decisions already settled (do not re-litigate)

- **Engine:** Meilisearch, self-hosted via Docker Compose alongside the existing Postgres
  (`docker-compose.yml`). No SaaS cost — explicit user constraint.
- **Source of truth stays Postgres.** Meilisearch is a **derived, disposable index** —
  always rebuildable from Postgres, never a primary store. Losing the index is a
  reindex, never data loss.
- **Sync strategy:** in-process NestJS `EventEmitter2` listeners on product
  create/update/delete fire immediate upserts/deletes into the index via a
  `SearchIndexerService`. A `@nestjs/schedule` job runs a periodic **full reindex** as a
  consistency safety net (catches missed/failed events, since there is no durable queue).
  **Explicitly NOT** using BullMQ/Redis for v1 — no durable-queue infra exists in the stack
  yet and the brief keeps infra minimal.
- **Vendor dedupe is OUT of scope for v1** (see Out of scope). 5 vendors selling
  "Vitamin C Serum" show as 5 separate listings.

## Scope (v1 — in)

1. **Meilisearch service** in Docker Compose (env-configured host/key, health check).
2. **Core full-text search** with typo tolerance.
3. **Field weighting:** title (`name`) > description > vendor name (`businessName`).
4. **Ranking beyond text match** (tie-breaking order, highest priority first):
   `in_stock` boost → `sales_velocity` → `vendor_rating` → `recency` → `price`.
   **See ranking-signal gaps below — some of these signals do not exist in the model yet.**
5. **Faceted filters:** category path, price range, vendor, attributes (brand, size, …).
6. **Autocomplete / suggest** endpoint — prefix search, smaller capped result set.
7. **Synonyms** for domain terms (moisturiser/moisturizer, SPF/sunscreen, brand aliases).

## Business rules it must honour (enforce, do not reinvent)

From [[Listing Types & Vendor Rules]], [[Public Storefront & SSR]], [[Category Taxonomy & Discovery]]:

- **Approved-vendor visibility is a hard rule.** Card #36 (`c2o6xfZs`) gates
  `GET /products` so only `platform` listings OR `vendor` listings whose vendor is
  `approved` are public. **The search index must honour the identical rule** — a
  pending/rejected/suspended vendor's product must never appear in a search result or a
  suggestion. Two acceptable implementations (pick one, see open questions):
  (a) never index non-approved vendors' products (and delete on suspension), or
  (b) index everything with a filterable `vendor_status` and always apply a
  `vendor_status = approved OR listing_type = platform` filter at query time.
  Vendor **status changes must reconcile the index** (approve → add, suspend/reject → remove).
- **Browse/search is public and SSR-safe.** The search + suggest endpoints are `@Public()`
  and must work with no token and server-side (`/shop` is `RenderMode.Server`).
- **Shared contract, never duplicated.** Result/suggestion/filter shapes live in
  `@hb/shared` (see contract impact). The API DTOs `implement` them.
- **Money is `numeric(12,2)` + explicit currency** — a search result carrying `price` must
  also carry `currency` (per [[Money & Currency Rules]]); price facets/sort must not assume
  a single currency across ZA/NA even under the 1:1 peg.
- **Listing-type invariant** — platform listings have no vendor; the search document's
  vendor fields must be nullable, mirroring `ProductDto.vendor?`.

## Search document design (target) vs. actual model — GAP ANALYSIS

The brief's illustrative document used field names that **do not match the real domain
model**. The index must be built from real DTOs (`ProductDto`, `VendorDto`, `CategoryDto`),
not the illustrative shape. Mapping:

| Illustrative field | Real source (verified) | Status |
|---|---|---|
| `title` | `Product.name` / `ProductDto.name` | rename → `name` |
| `description` | `Product.description` | OK |
| `category_path` (array) | `Category` is self-referencing (`parentId`), **no materialized path exists** | **GAP — must be derived** |
| `vendor_name` | `Vendor.businessName` / `ProductVendorDto.businessName` | rename → `businessName` |
| `vendor_id` | `Product.vendorId` (nullable — absent on platform) | OK (nullable) |
| `vendor_rating` | **does not exist** on `Vendor`/`VendorDto` | **GAP — no rating in model** |
| `price` | `Product.price` (`numeric(12,2)`) + `currency` | OK, **add `currency`** |
| `attributes` (color/size/brand) | **does not exist** — products have no attributes/variants model | **GAP — no attributes in model** |
| `in_stock` | `Product.stockQuantity` (int) → derive `inStock = stockQuantity > 0` | derive |
| `sales_count_30d` | **does not exist** — no order-rollup / sales metric on products | **GAP — no sales signal** |
| `created_at` | `Product.createdAt` | OK (→ recency) |
| `tags` | **does not exist** on `Product` | **GAP — no tags field** |

**Consequence:** three of the five ranking signals (`sales_velocity`, `vendor_rating`,
`recency`, plus `in_stock`) and the `attributes` facet are **not fully backed by the current
schema**. `in_stock` and `recency` are derivable today (`stockQuantity`, `createdAt`).
`sales_velocity`, `vendor_rating`, `attributes`, `category_path`, and `tags` are **not**.
See open questions — a human must decide scope before these cards ship. The recommended v1
cut is below.

### Recommended v1 document (buildable from today's schema, no new domain features)

```
id                 Product.id
name               Product.name                 (searchable, weight 1 — highest)
description        Product.description          (searchable, weight 2)
businessName       Vendor.businessName | null    (searchable, weight 3; null on platform)
vendorId           Product.vendorId | null
vendorStatus       Vendor.status | 'platform'    (filterable — approved-only gate)
listingType        Product.listingType           (filterable)
price              Product.price                 (filterable range + sortable)
currency           Product.currency              (returned; do not sort across currencies)
categoryIds        Category.id[]                 (filterable facet)
categoryNames      Category.name[]               (searchable low weight + facet)
inStock            stockQuantity > 0             (filterable + rankingRule boost)
stockQuantity      Product.stockQuantity         (returned)
originCountry      Product.originCountry         (filterable)
createdAt          Product.createdAt (unix)      (rankingRule: recency desc)
imageUrl           primary ProductImage.url|null (returned for result cards)
```

`sales_velocity`, `vendor_rating`, materialized `category_path`, `attributes`, and `tags`
are **deferred to their own cards / future phases** (see open questions + Out of scope) so
v1 ships against the real schema without inventing domain features.

## @hb/shared contract impact

**Existing (do not collide):** `libs/shared/src/contracts/search.ts` already defines the
storefront **omnibox** suggest shape — `SearchSuggestQuery`, `ProductSuggestion`,
`VendorSuggestion`, `CategorySuggestion`, `SearchSuggestions`. This is a grouped
product+vendor+category omnibox, distinct from the engine's product-focused search.
**Decide (open question):** extend/reuse this file vs. add a parallel engine namespace.
Recommendation: keep the omnibox types, add engine types alongside them in the same file.

**New interfaces (v1):**

- `ProductSearchQuery` — `{ q?: string; categoryId?: string; vendorId?: string;
  minPrice?: number; maxPrice?: number; inStockOnly?: boolean; page?: number;
  pageSize?: number; sort?: ProductSearchSort }`.
- `ProductSearchSort` — enum-like const object (`relevance | priceAsc | priceDesc | newest`).
  New enum file `libs/shared/src/enums/product-search-sort.ts` + export from `enums/index.ts`.
- `ProductSearchResultItem` — one hit: `{ id, name, price, currency, inStock, listingType,
  vendorId?, businessName?, imageUrl, categoryIds, createdAt }`.
- `ProductSearchFacets` — returned facet counts: `{ categories: Array<{id,name,count}>;
  vendors: Array<{id,name,count}>; priceRange: { min: number; max: number } }`.
- `ProductSearchResponse` — `{ items: ProductSearchResultItem[]; total: number; page: number;
  pageSize: number; facets: ProductSearchFacets; query: string }`.
- `ProductSuggestQuery` / suggest response — either reuse existing `SearchSuggestions` or a
  new lean `{ suggestions: Array<{ id; name; imageUrl }> }` (open question — align with the
  existing omnibox rather than duplicate it).

All query inputs get API DTO classes (class-validator) that `implement` these interfaces,
per the non-negotiable "every endpoint input is a validated DTO implementing a shared interface".

## Sync architecture (no queue)

- **Write path:** `ProductsService` create/update/delete and `VendorsService` status change
  emit `EventEmitter2` events (`product.created|updated|deleted`, `vendor.status.changed`).
  `SearchIndexerService` listens and upserts/deletes the affected document(s) in Meilisearch.
  Vendor approval/suspension reconciles all that vendor's product documents.
- **Safety net:** a `@nestjs/schedule` cron runs **daily (overnight)** — resolved 2026-07-05,
  see open question 7 — doing a **full reindex** from Postgres, the authoritative rebuild path
  that also serves cold start and index-loss recovery. Must be idempotent and safe to run
  while the API serves traffic.
- **No durable queue (v1):** events are best-effort in-process. The scheduled full reindex is
  the reconciliation guarantee. If an event is lost (process crash mid-write), the next
  reindex heals it — bounded staleness, acceptable for v1. Durable queue (BullMQ/Redis) is a
  future upgrade if staleness bounds tighten.
- **Meilisearch config** (settings applied on boot / reindex): searchable attributes +
  order (name > description > businessName > categoryNames), filterable attributes
  (categoryIds, vendorId, vendorStatus, listingType, price, inStock, originCountry),
  sortable (price, createdAt), ranking rules (custom order for the in_stock → recency
  business ranking), and the synonyms map (loaded from the admin-editable Postgres source,
  see card #49).

## Out of scope (v1)

- **Canonical-product / buy-box grouping (vendor dedupe).** Near-duplicate products from
  different vendors show as separate listings. Grouping many vendor offers under one
  canonical product (one result, multiple offers, a "buy box") is an **explicitly deferred
  future phase**. Backlog card created as a placeholder — NOT part of the v1 batch.
- **`vendor_rating`** — no rating model exists; a ratings/reviews feature is its own epic.
  Ranking rule reserved but inert until it lands.
- **`sales_velocity` / `sales_count_30d`** — no sales-rollup metric exists. Needs an
  order-aggregation feature first. Ranking rule reserved but inert until it lands.
- **`attributes` facet (brand/size/color)** — no product attribute/variant model exists.
  Faceting on attributes waits on a product-attributes feature.
- **Materialized `category_path`** — v1 facets on `categoryIds`/`categoryNames`; a full
  breadcrumb path array (ancestors) is a nice-to-have once category depth grows.
- **~~Admin-editable synonyms~~ — NOW IN v1.** Resolved 2026-07-05: synonyms are
  admin-editable (Postgres-backed entity + admin CRUD endpoint + reload into Meilisearch),
  no static checked-in config file. See open question 9 and card #49.
- **BullMQ/Redis durable queue**, multi-language search, personalization/learning-to-rank,
  frontend search UI components (this spec is API + engine + sync; the storefront search box
  is a separate web card once the API lands).

## Open questions (confirm before /ship-card)
All open questions are now **resolved** (decisions taken 2026-07-05). Recorded here so the
note reads as settled; cards updated to match.

1. **Missing ranking signals — cut or stub?** RESOLVED — **cut to `inStock → recency → price`
   for v1.** `sales_velocity`/`sales_count_30d` and `vendor_rating` have no backing data in
   the schema (no sales-rollup, no ratings model). Their two ranking slots are reserved as
   **no-ops in the Meilisearch ranking config**, to be wired in once that data exists
   elsewhere. Not blocking v1. (Card #48.)
2. **Attributes facet?** RESOLVED — **dropped from v1** (no attribute/variant model). v1
   facets on category + vendor + price only. Revisit when a product-attributes feature lands.
3. **`category_path`?** RESOLVED — **index flat `categoryIds` + `categoryNames` for v1**;
   materialized ancestor-path arrays deferred until category depth warrants it.
4. **Approved-vendor gate — index-time exclude vs. query-time filter?** RESOLVED —
   **query-time filter.** Every search/suggest query applies
   `vendorStatus = approved OR listingType = platform` at query time; products are NOT
   excluded at index time. Chosen for simplicity/auditability and because vendor status
   changes take effect instantly without the indexer reacting to every status change. The
   index stays a faithful mirror of Postgres. (Cards #47, #48, #50.)
5. **Engine search supersede or coexist with `ILIKE` `GET /products?q=`?** RESOLVED —
   **coexist initially**; new `/search` + `/search/suggest` endpoints ship first, storefront
   migrates to them, then the `ILIKE` path is retired in a later cleanup card.
6. **Reuse existing `SearchSuggestions` omnibox contract for suggest, or new lean shape?**
   RESOLVED — **keep the omnibox contract**; the engine's suggest returns a lean
   product-only shape aligned with it. No duplication of the omnibox types.
7. **Full-reindex cron interval?** RESOLVED — **daily, overnight.** In-process event
   listeners rarely miss updates, so drift risk is low; a daily overnight full reindex is the
   safety net while minimizing load. (Not hourly.) (Card #47.)
8. **Meilisearch persistence + secrets?** RESOLVED — index data dir mounted as a **named
   Docker volume** so the index survives container restarts (no full reindex on every
   recreate). Master API key supplied via **env var only** (`.env`, never committed;
   `.env.example` reference), consistent with other secrets in this repo. (Card #45.)
9. **Synonyms: static config vs. admin-editable?** RESOLVED — **admin-editable.** Synonyms
   (moisturiser/moisturizer, SPF/sunscreen, brand aliases) are stored in **Postgres** and
   edited via an **admin endpoint/UI without a deploy** — NOT a static checked-in file. This
   is a bigger scope than the original static-file sketch: it needs a synonyms table/entity +
   migration, an admin CRUD endpoint with a validated DTO implementing a shared interface,
   permission-gating, and a mechanism to push updated synonyms into Meilisearch (on save
   and/or via the reindex settings path). (Card #49 — scope expanded accordingly.)

## Vertical slices → Trello cards

| # | Title | Layer | Card |
|---|---|---|---|
| 1 | SearchModule scaffold + Meilisearch in Docker Compose | infra + api | #45 `sI1Vaxp0` |
| 2 | @hb/shared search contracts + validated DTOs | shared + api | #46 `dhZ1PEDE` |
| 3 | SearchIndexerService: event listeners + scheduled full reindex | api | #47 `NbqwbivJ` |
| 4 | SearchService: ranking, filters, facets, suggest | api | #48 `jphU7kkC` |
| 5 | Domain synonyms config + load into Meilisearch | api | #49 `ig9t1BQQ` |
| 6 | Search API endpoints: /search + /search/suggest | api | #50 `eZnGUtey` |
| 7 | [BACKLOG — deferred] Canonical-product / buy-box grouping | future | #51 `Ha9AMFWV` |

Recommended order: 1 → 2 → (3, 5 parallel) → 4 → 6. Card 7 is backlog, not this batch.


## Implementation Notes

### Card #53 — Search: admin synonyms UI (Angular) — Slice 5c

**What shipped (2026-07-17):** Angular admin screen `admin-search-synonyms` for managing search synonyms — the UI layer on top of card #52 CRUD endpoints. Admins can now list/create/edit/delete synonym groups without direct API calls (delivers the "no deploy required" goal).

**Where:**
- `apps/web/src/app/core/api/admin-search-synonyms.service.ts` — typed HttpClient service (`list`/`create`/`update`/`remove`) against `GET/POST /api/admin/search/synonyms` and `PATCH/DELETE /api/admin/search/synonyms/:id`, consuming `@hb/shared` `SynonymDto`/`SynonymCreateRequest`/`SynonymUpdateRequest` (no duplicated models).
- `apps/web/src/app/features/admin/pages/admin-search-synonyms/` — standalone, signals-based page; typed reactive form with a `FormArray` of equivalents (add/remove, cap 20), shared create/edit form, inline per-row delete confirmation (SSR-safe, no `window.confirm`).
- Route `admin/search-synonyms` under the existing role-guarded admin block; new "Synonyms" nav item in `admin-shell`.

**Key decisions:**
- Form validation mirrors the server DTO (term required + max 100; ≥1 equivalent; each required + max 100; no duplicates — case/whitespace-insensitive) surfaced as inline field errors.
- Term + equivalents trimmed/lowercased client-side on submit to match server normalization.
- Followed the existing `admin/pages/*` idiom (hand-rolled HTML + DESIGN.md tokens + material-symbols) rather than Angular Material, for consistency with sibling admin pages (admin-users/admin-orders).
- Equivalents stored as `FormArray` with 20-item cap, enforced client-side.
- Delete confirmation inlined (safe for SSR), no `window.confirm`.

**No contract or schema change** — consumed existing `@hb/shared` synonym contracts; UI-only.

**Tests/review:**
- Vitest specs for service + page (load, create/edit/delete happy paths, validation errors, load-error); full web suite green (446 tests).
- `npm run build` clean; code-review verdict SHIP (one a11y nit — per-input aria-labels — fixed).

**Follow-ups:** none. (Related backlog: card #51 canonical-product/buy-box grouping remains deferred.)