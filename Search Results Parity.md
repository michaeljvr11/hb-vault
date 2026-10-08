# Search Results Parity

Front-of-funnel spec (2026-10-08). Implementation flows through `/ship-card` — no code here.
Related: [[Product Search Engine]] · [[Category Taxonomy & Discovery]] · [[Public Storefront & SSR]] · [[Listing Types & Vendor Rules]] · [[Money & Currency Rules]].

Raised by Josh in QA: the suggestion box finds products, but pressing Enter shows none.

## Problem

Type a synonym (admin-defined, e.g. "sunscreen" for SPF) or a category name ("clothing") into the
search bar and the dropdown lists the matching products. Press Enter and the `/discover` results
page is empty.

## Diagnosis — two different engines behind one search box

| Surface | Route | Engine | What it matches |
|---|---|---|---|
| Dropdown (`Products` group) | `GET /search/suggest` → `ProductSearchService.suggest` | **Meilisearch** | typo-tolerant prefix match on `name`, `description`, `businessName`, `categoryNames`, plus the admin synonyms |
| Results page (`/discover?q=`) | `GET /products?q=` → `ProductsService.findAll` | **Postgres `ILIKE`** | the literal substring on `product.name` / `product.description` only |

Enter navigates to `/discover?q=…` (`search-bar.ts` Enter → `Discover.onSearchSubmit`; the header bar and the
mobile bar do the same via `nav-bar.ts`). `Discover.fetchProducts` calls `ProductsService.list` → `GET /products`.
**`Discover` has never been wired to `/search`.** So:

- **Synonyms** are applied only inside Meilisearch. `ILIKE '%sunscreen%'` never sees "SPF", so zero rows.
- **Category names** are indexed as `categoryNames` in Meilisearch, but `findAll`'s `q` never looks at categories, so
  "clothing" matches only products that literally say "clothing" in their name or description.
- Typos and word order fail in the same way for the same reason.

Clicking a *Categories* row in the dropdown works because it takes a different path (`?categoryId=`, which is an
exact join, not text search). That is why the category "appears to work" until you press Enter.

This is the unfinished tail of [[Product Search Engine]] open question 5: *"coexist initially; storefront migrates to
`/search` + `/search/suggest`, then the ILIKE path is retired in a later cleanup card."* The suggest half migrated (card
#50); the results half never did, and no card tracked it. `origin/main` and the Trello board were checked on 2026-10-08:
nothing in flight covers it.

### Second, smaller gap found on the way

The index is kept fresh by product and vendor events only (`SearchIndexerService`). `CategoriesService.update` emits
nothing. Rename "Clothing" to "Apparel" and every product's indexed `categoryNames` stays "Clothing" until the 03:00
full reindex, so category-name search would be wrong for up to a day. Deletion is not affected (`remove` refuses a
category that still has products). Today this barely matters, because the results page ignores `categoryNames`; once
results are engine-backed it does.

## Decisions to confirm (recommended in bold; cards are written to these)

1. **Where to fix it — server-side behind `GET /products?q=`.** **Recommended:** when `q` is non-empty, `ProductsService.findAll`
   asks the Meilisearch engine for the ranked, filtered page of product ids, then hydrates the full `ProductDto` from
   Postgres in that order. With no `q`, nothing changes. This repairs every caller of `?q=` at once (header bar, mobile bar,
   `/discover`, vendor storefront drill-down) and keeps `ProductDto`/`PagedResponse` as the one result contract, so the
   product card (rating, sizes, vendor, images) needs no mapping layer.
   *Rejected:* pointing `Discover` at `GET /search`. `ProductSearchResultItem` lacks `averageRating`, `reviewCount`, `sizes`
   and `images`, so the web would need a second card model or the index would need fast-changing rating data; every caller
   would also have to migrate separately. `GET /search` stays as the faceted engine endpoint for a future filter UI.
2. **If Meilisearch is down: fall back to the current `ILIKE` query, log a warning.** **Recommended.** The index is
   disposable and search is the main way in; a degraded result beats a 500. (The alternative is to fail the request.)
3. **Sorting.** `ProductSort` gains `relevance`. **Recommended:** when `q` is set and the URL has no explicit `?sort=`, the
   page uses `relevance`; with no `q` the default stays `newest`. An explicit `newest` / `price_*` / `name` is honoured.
   Without this, `Discover`'s hard default of `newest` would override the engine's ranking and make the new results look
   random.
4. **Accepted behaviour change.** Meilisearch matches words, prefixes and near-typos, not arbitrary substrings. A search
   for "shirt" will no longer return "Tshirt". The dropdown already behaves this way, so the two now agree.

## Synonyms for vendors and categories (added 2026-10-08, confirmed by Michael)

Admin-configured synonyms must reach the **Vendors** and **Categories** rows of the dropdown too, not just products.
Example: an admin saves the group *clothes <-> clothing*; typing "clothes" must list the "Clothing" category row.

Why it needs its own work: those two groups are not in Meilisearch. `SearchService.suggestVendors` /
`suggestCategories` are Postgres `ILIKE '%q%'` queries, and the admin-editable synonym table
(`Synonym` entity, `SynonymsService`) is only ever pushed into the Meilisearch index settings via
`buildMeilisearchSynonymsMap`. So the synonym map has one consumer today, and the two Postgres groups need a second.

Approach (SRCH-5):
- Reuse the **same** `buildMeilisearchSynonymsMap` output (one mapper, no drift): it already encodes `enabled`,
  `bidirectional` and sibling equivalents. Do not reimplement the rules.
- Expand the typed term into itself plus its mapped equivalents, then match any of them with `ILIKE` on
  `category.name` and on `vendor.businessName` / `vendor.tradingName`. Approved-only vendor filter and the 5-per-group cap are unchanged.
- Match the whole typed term (trimmed, case-insensitive) against a synonym key, the same way the engine treats it.
  Prefix-of-a-key matching while mid-word ("clothe" -> clothing) is **not** included; the engine does not do it either.
- Keep an in-process copy of the map so a suggest request (one per debounced keystroke) does not hit the synonym table
  every time; invalidate it on admin create/update/delete, with a short TTL as a safety net.
- Bound the expansion (a small cap on terms) and escape `%`, `_` and `\` in every term, so a typed `%` cannot match everything.
- Admin screen copy states that synonyms apply to products, vendors and categories.

Enter-key results are already covered: product search (SRCH-1) matches `categoryNames` and `businessName` through the
engine, whose synonyms the admin already controls.

## Business rules it must honour

- **Approved-vendor visibility** ([[Listing Types & Vendor Rules]], card #36). The engine filter already applies
  `vendorStatus = approved OR listingType = platform`. The Postgres hydration step **re-applies the same visibility
  predicate** because Postgres is the source of truth and the index can lag behind a lost event. A stale id never leaks.
- **Money.** Results keep `price`, `currency`, `effectivePrice`, `isDiscountActive` exactly as `ProductDto` today. A price
  sort under a text query groups by currency first (the engine's existing `currency:asc` rule) so ZAR and NAD are never
  interleaved; the ZAR/NAD peg stays data.
- **SSR-safe, public.** `GET /products` stays `@Public()`; `/discover` stays server-rendered and the HTTP transfer cache is
  untouched (no new origin, no new route).
- **Paging.** `total`, `page`, `limit` keep their meaning. Engine totals are capped at Meilisearch's `maxTotalHits`
  (default 1000), which is far above any realistic page range for this catalogue.

## @hb/shared contract impact

- `ProductSort` (`libs/shared/src/contracts/product.ts`): `'newest' | 'price_asc' | 'price_desc' | 'name'` gains
  `'relevance'`. Doc the semantics: "text-relevance order; only meaningful with `q`, behaves as `newest` otherwise".
- `ProductQueryDto` (`PRODUCT_SORT_VALUES`) accepts it.
- No new interfaces, no change to `ProductDto`, `PagedResponse`, `SearchSuggestions` or `ProductSearch*`.
- No schema change, no migration. One Meilisearch settings change if `name` sort is kept under a query (see SRCH-1).

## Slices → Trello cards

| # | Title | Layer | Depends on | Card |
|---|---|---|---|---|
| SRCH-1 | Back `GET /products?q=` with the Meilisearch engine | shared + api | — | `F4OMk15F` |
| SRCH-2 | Discover: relevance sort when a term is searched | web | SRCH-1 | `lVX3EMus` |
| SRCH-3 | Opt-in suggest-vs-results parity check against a live stack | api tooling | SRCH-1 | `GR9GDcEn` |
| SRCH-4 | Refresh a category's products in the index when it is renamed | api | — (do before or with SRCH-1) | `GAqT32Hx` |
| SRCH-5 | Admin synonyms apply to the Vendors and Categories dropdown rows | api + admin copy | — | `GLIk2ojj` |

Order: SRCH-4, SRCH-1 and SRCH-5 are independent and can run in parallel → SRCH-2 → SRCH-3. SRCH-1 + SRCH-2 are a natural pair for one `/ship-batch`
(one branch, one PR); SRCH-4 is independent and small.

## Out of scope

- A facet/filter UI on `/discover` (price range, vendor, category facets). `GET /search` already supports them; separate card when wanted.
- Typo tolerance for the *Vendors* and *Categories* dropdown rows. They stay Postgres `ILIKE`; only **synonyms** are added
  to them (SRCH-5). Typos on those two groups are not addressed.
- Retiring the `ILIKE` branch. It stays as the outage fallback (decision 2).
- Canonical-product / buy-box grouping (still deferred, #51).
- Real ranking signals (`sales_velocity`, `vendor_rating`) — still reserved no-ops.

## Open questions

1. Decisions 1–3 above: confirm or override before `/ship-card`.
2. `name` sort under a query: add `name` to the index's `sortableAttributes` (settings are re-applied on boot, no migration),
   or hydrate the capped hit set and sort it in Postgres. SRCH-1 recommends the sortable attribute but must verify that
   Meilisearch's string ordering is case-insensitive enough to match what shoppers expect; if not, fall back to the
   Postgres sort for that one case.
