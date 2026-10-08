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
  Matching while the shopper is still mid-word ("clothe" -> clothing) is delivered by SRCH-6 through the **same map**
  (derived prefix keys), so this helper needs no prefix logic of its own.
- Keep an in-process copy of the map so a suggest request (one per debounced keystroke) does not hit the synonym table
  every time; invalidate it on admin create/update/delete, with a short TTL as a safety net.
- Bound the expansion (a small cap on terms) and escape `%`, `_` and `\` in every term, so a typed `%` cannot match everything.
- Admin screen copy states that synonyms apply to products, vendors and categories.

Enter-key results are already covered: product search (SRCH-1) matches `categoryNames` and `businessName` through the
engine, whose synonyms the admin already controls.

## Synonyms should fire while the shopper is still typing (added 2026-10-08, SRCH-6)

**Ask.** Product "Tennis Balls", admin group *tennis balls <-> racket balls*. Typing "tennis" finds it. Typing "racket"
(or "racket b") does not; it only appears once the full phrase "racket balls" is typed. The dropdown is search-as-you-type,
so the synonym should work on a partial entry too.

**Research (Meilisearch v1.15.2, the version prod pins; verified against a throwaway container, not the dev index).**
- Meilisearch already treats the *last* query word as a prefix, but it looks synonyms up only for a **complete** term of
  1-3 words that equals a synonym key. The docs are silent on prefixes; the behaviour below was measured.
- Measured: with the group above, `tennis balls` returns the product, while `tennis b` / `tennis ba` do not, and
  `racket` / `racket b` / `racket ba` do not return a "Tennis Balls" product either. Same for `clothe` vs `clothes`.
- Docs facts that constrain the design: synonyms are fetched only for search terms of 1-3 words; the literal query always
  outranks its synonyms; one-way synonyms stay one-way.

**Options weighed.**
1. **Derive prefix keys into the synonym map (chosen).** `buildMeilisearchSynonymsMap` also emits a key for each
   prefix of every key (min length 3 characters, keys of at most 3 words), pointing at the same equivalents. Measured:
   after adding them, `racket`, `racket b` and `racket ba` all return "Tennis Balls". Cost: about 9 extra keys per 12-char
   term (2 groups -> 18 keys; roughly 1,000 keys at 100 groups), which Meilisearch handles easily and which are never stored
   in Postgres or shown in the admin screen. It fixes products, and (through SRCH-5's helper reading the same map)
   vendors and categories, in one place. Sorting, filters, facets and paging keep working because it is still one plain query.
2. **Query-time expansion with Meilisearch federated multi-search (rejected).** Search the typed text plus the equivalents of
   any key it prefixes, merged by relevance. It works for relevance order, filters and offset/limit, but the merge **ignores
   per-query `sort`** (measured: `createdAt:desc` came back in relevance order), so price and newest sorts would be wrong,
   and it costs extra engine calls and code in three places.

**Known trade-offs (state them, do not hide them).**
- Typing the first word of a multi-word key ("racket") now also surfaces products matching its equivalents ("tennis balls").
  That is the ask, but it is a recall-over-precision choice.
- Literal matches still rank high but synonym-only matches can interleave with them, and a very short prefix ("rac") can rank
  the synonym match first. A minimum prefix length of 3 is the recommended floor; 4 is stricter (open question below).
- The dropdown shows at most 5 products; if 5 literal matches exist, a synonym-only product shows on Enter, not in the box.

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

| SRCH-6 | Synonyms fire while the shopper is still typing (derived prefix keys) | api | — (SRCH-5 picks it up for free) | `g56dzKWt` |

Order: SRCH-4, SRCH-1, SRCH-5 and SRCH-6 are independent and can run in parallel → SRCH-2 → SRCH-3. SRCH-1 + SRCH-2 are a natural pair for one `/ship-batch`
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
3. SRCH-6 minimum prefix length: 3 characters (more forgiving, more noise) or 4 (stricter). Recommendation: 3, kept as one
   named constant so it can be tuned from real catalogue data after launch.


## Implementation Notes (2026-10-08)

**Shipped:** one bundled branch `feat/F4OMk15F-search-results-parity`, one PR. Commits: `6d2a011` SRCH-6, `f507042` SRCH-1, `cf5b243` SRCH-5, `4ae493c` SRCH-2, `3f1c4b0` review fixes. **PR:** [#119](https://github.com/michaeljvr11/hb-mono-repo/pull/119). Cards: [SRCH-1](https://trello.com/c/F4OMk15F), [SRCH-2](https://trello.com/c/lVX3EMus), [SRCH-5](https://trello.com/c/GLIk2ojj), [SRCH-6](https://trello.com/c/g56dzKWt). SRCH-3 and SRCH-4 are not in this batch.

### SRCH-1 — `GET /products?q=` on the engine

- `ProductsService.findAll` with a non-blank trimmed `q` calls `ProductSearchService.searchProductIds`. Meilisearch is paged with `page` / `hitsPerPage`, so `total` is the exact engine count, capped by `maxTotalHits` (1000).
- One Postgres `find` re-applies the platform OR approved-vendor predicate, reorders rows to engine rank and drops ids that are missing. `total` is the engine count, so a page can be shorter than `limit`.
- Only the engine call falls back to ILIKE, with a warning. Hydration errors surface. The ILIKE fallback now escapes `%`, `_` and `\`.
- `ProductSort` gained `relevance`. Without `q` it behaves as `newest`. An omitted `sort` with `q` means `newest` server-side, so the web sends `sort=relevance`.
- Open question 2 (name sort) resolved: `name` is in the index `sortableAttributes` (one line in `6d2a011`). Checked on a throwaway Meilisearch v1.15 container: `name:asc` is case-insensitive and accent-folded (`10 pack, 2 pack, Apple, apricot, banana, cherry, eagle, Émile, Zebra`).
- Settings are re-applied on API boot. Each environment needs one restart (or reindex) before `sort=name` with `q` works. Until then Meilisearch rejects the sort and the request falls back to ILIKE with a warning.
- Known inconsistency: engine price sorts group by currency, then `effectivePrice`. The q-less and ILIKE path sorts on base `price` with no currency grouping.

### SRCH-6 — derived prefix synonym keys

- `buildMeilisearchSynonymsMap` emits a key for each prefix of every key. Constants in `search.constants.ts`: `MIN_SYNONYM_PREFIX_LENGTH` = 3, `MAX_SYNONYM_KEY_WORDS` = 3, `SYNONYM_MAP_WARN_KEYS` = 5000.
- Real keys win over derived ones. Output is sorted, so it is deterministic. The mapper collapses inner whitespace and sorts equivalents.
- `applySettings` logs a warning above 5000 keys and applies the map anyway.
- The map is `Map`-based, so terms such as `constructor` and `__proto__` work (review fix in `3f1c4b0`).
- Open question 3 resolved: 3 characters, confirmed by Michael.
- **Trade-off (raised in review, accepted):** derived prefixes also fire for finished short words. `carpet <-> rug` makes `car` return rugs; `red bush <-> rooibos` makes `red` return rooibos. Meilisearch cannot tell a half-typed word from a finished one. Intended per spec. Tune `MIN_SYNONYM_PREFIX_LENGTH` from real catalogue data after launch.

### SRCH-5 — admin synonyms on the Vendors and Categories rows

- New `SynonymExpansionService`. It reads the same mapper output through `SynonymsService.getCachedMeilisearchSynonymsMap`, so there is one mapper.
- In-process cache with a 60 s TTL (`SYNONYM_MAP_CACHE_TTL_MS`). A generation counter stops an in-flight load from overwriting a newer invalidation. Admin create, update and delete invalidate the cache before the Meilisearch reload. Other API instances see an edit only after the TTL.
- Term cap 10 per expansion. Terms are matched with `ILIKE ANY(patterns)` on `category.name`, `vendor.businessName` and `vendor.tradingName`. `%`, `_` and `\` are escaped, so a typed `%` no longer matches every row.
- If the synonym lookup fails, results fall back to un-expanded. The approved-only vendor filter and the 5-per-group cap are unchanged.
- Admin screen copy says synonyms apply to products, vendors and categories.
- Partial typing on these rows comes from SRCH-6's derived keys. No prefix logic lives in this helper.

### SRCH-2 — Discover relevance default

- With `q` set and no `?sort=`, `/discover` uses `relevance`. With no `q`, the default stays `newest`.
- A stale `?sort=relevance` without a term is treated as `newest`.
- Choosing Relevance drops the `sort` param. Clearing the term, or taking a Categories suggestion, drops a dead `sort=relevance`.
- The Relevance option is shown only when a term is present.

### Live verification (dev stack, 2026-10-08)

- `redbush`, `redb`, `red bu` and `red bush t` each return "HB Rooibos & Honeybush Gift Set" from both `GET /products?q=` and `GET /search/suggest` (existing group rooibos <-> redbush, red bush tea).
- Category-name search (`beauty`) and a typo (`ceramc`) return products on Enter.
- Temporary groups: Categories and Vendors rows expand by synonym, including partial typing (`zz-cosm` -> Health & Beauty), case-insensitively. A typed `%%` matches no vendor or category row. The temporary rows were deleted afterwards.
- The SRCH-6 "tennis balls" check used the existing rooibos group, not a new product.
- ILIKE ANY was verified on real Postgres, not only in the fake query builder.
- Discover checked at phone width and 1280 px: no hydration warnings. Only the expected anonymous `/api/auth/refresh` 401.

### Tests at ship time

- api 101 suites / 1781 tests. web 108 files / 1673 tests. `lint:api` clean. Build green (web build needs `INTERNAL_API_BASE_URL` set).

### Not done / follow-ups

- SRCH-3 (parity script, `GR9GDcEn`) not shipped. Its defaults for `racket`, `racket b` and `racket ba` are still to add.
- SRCH-4 (category rename reindex, `GAqT32Hx`) not in this batch. A renamed category keeps stale indexed `categoryNames` until the 03:00 reindex, so category-name results can lag.
- Reviewer's optional cleanups, not taken: fold `SynonymExpansionService` into `SearchService`; drop the `dropRelevanceSort` helper in `discover.ts`.
