---
type: spec
tags:
  - storefront
  - shop-home
  - ui
---

# Shop Home Sections

Status: **SHS-1 to SHS-7 and the radial nav follow-up shipped 2026-10-26.** SHS-1 and SHS-2 merged ([PR #127](https://github.com/michaeljvr11/hb-mono-repo/pull/127)). SHS-3, SHS-4 and SHS-5 merged ([PR #128](https://github.com/michaeljvr11/hb-mono-repo/pull/128)). SHS-6, SHS-7 and the radial nav scale in [PR #129](https://github.com/michaeljvr11/hb-mono-repo/pull/129), awaiting human merge; #129 replaces the client-side shelf derivations from #128.
Related: [[Public Storefront & SSR]] · [[Product Discounts & Sale Pricing]] · [[Product Reviews & Ratings]] · [[Listing Types & Vendor Rules]] · [[Category Taxonomy & Discovery]] · [[Storefront UI Cleanup Batch]] · [[Product Image Optimization Pipeline]]

## Problem

The `/shop` home page shows products in exactly one place: the "New in Namibia" carousel (the 8 newest
products). Everything else on the page is a category tile, a vendor tile or marketing copy, so a visitor
who is not interested in the newest 8 items has nothing to browse without leaving for `/discover`.
Separately, the hero band is too tall (measured by CSS: ~560px at ≥1280, 600px at ≥1536 and at md; below
xl it is content-driven), so on a laptop the first product row starts well below the fold.

## Scope

1. **Hero** — keep the photograph and copy, make the band slightly shorter (**SHS-1**).
2. **More product sections** on `/shop`, each a horizontal shelf identical in look to "New in Namibia":
   - **On sale** — products with an active discount (**SHS-3**)
   - **Top rated** — highest `averageRating` (**SHS-3**)
   - **Made in Namibia** and **From South Africa, delivered to Namibia** — split by `originCountry` (**SHS-4**)
   - **Category shelves** — the 2 categories with the most products, a shelf each (**SHS-5**, optional)
3. A reusable **`ProductShelf`** component extracted from the existing section so the above are not
   five copies of the same markup (**SHS-2**, a pure refactor; "New in Namibia" must not change).

## Research findings (what the code and vault already say)

- `/shop` makes one products request, `GET /products?limit=100` (the server max, `MAX_LIMIT = 100`,
  newest first), and slices the first 8 for the carousel (`shop.ts` `CAROUSEL_LIMIT`). Category counts are
  already derived client-side from that same list (`deriveCategoryCounts`). So the list needed for every
  new shelf is **already in memory** — no new request, no new endpoint. *Superseded by SHS-6 (PR #129): the page makes one `GET /products/shelves` request and the category counts come from `CategoryDto.productCount`.*
- `ProductDto` already carries everything these shelves need, server-computed: `isDiscountActive`,
  `effectivePrice`, `discountPercent`, `averageRating`, `reviewCount`, `originCountry`, `listingType`,
  `categories`, `stockQuantity`, `sizes`. **`@hb/shared` impact: none.**
- `ProductQuery` supports only `categoryId`, `q`, `vendorId`, `page`, `limit`, `sort` (`relevance | newest |
  price_asc | price_desc | name`). There is no `onSale` filter and no rating sort. Adding them is the
  scale-up path (see Open questions), not needed today. *Superseded by SHS-6: `onSale`, `inStock`, `originCountry`, `minRating` and the rating and discount sorts were added.*
- **"Best sellers" is not buildable.** There is no sales/popularity signal in the data model. The same
  conclusion is already recorded for vendor storefronts (VPC-6 in [[Vendor Profile Customization]],
  [[Product Reviews & Ratings]]). It needs a sales-signal model first; out of scope here.
- Rate limit: the API throttles at 120/min **per (route, IP)** and SSR forwards the real client IP.
  Reusing the one products request keeps the page at its current 4 API calls.
- Positioning/legal: after `cc1eaec` the site leads with "Namibia's marketplace" and **must not claim
  duty-free or prepaid customs** — duty and VAT may apply to cross-border orders. Origin-shelf copy must
  respect that (no "tax-free", "duty-free", "customs cleared").
- Discount display rule ([[Product Discounts & Sale Pricing]]): a display surface renders `effectivePrice`
  / `isDiscountActive`, **never recomputes** the sale price from `discountPercent`. `ProductCard` already
  does this; shelves reuse the card, so this is inherited, not re-implemented.
- Ratings rule ([[Product Reviews & Ratings]]): `averageRating`/`reviewCount` are **absent (not 0)** when
  a product has no reviews. A top-rated derivation must treat absent as "not rated", never as 0.
- Wishlist hydration gate: hearts render empty until `afterNextRender` (avoids an SSR/client mismatch).
  Every shelf must route wishlist state through the page's existing `isWishlisted` gate.

## Behaviour (rules the cards must honour)

- *Superseded by SHS-6 (PR #129): shelves are built server-side in one `GET /products/shelves` response, and the web does no shelf derivation.* Original rule: all new shelves are **derived client-side from the already-loaded product list** by pure, exported,
  unit-tested functions (same pattern as `deriveCategoryCounts`).
- **Max 8 products per shelf; a shelf renders only with at least 4 products.** Below that the whole
  section is omitted (no empty state, no heading) — a half-empty shelf looks broken.
- **One appearance per page** (open question 3, decided 2026-10-26). Shelves are filled in page order; a product already shown is skipped in later shelves, and a shelf left under the minimum is omitted, which frees its products.
- While products are `loading`/`error`/`empty`, only the existing "New in Namibia" section shows its
  skeleton / error / empty state. Extra shelves render nothing (no repeated error cards, no phantom
  skeletons). Under SSR the data is fetched server-side so shelves are in the first HTML.
- Curated shelves (On sale, Top rated, origin, category) **exclude out-of-stock products** (total stock
  = `stockQuantity`, or the sum of `sizes[].stockQuantity` when sized — same rule as `product-card.ts`).
  "New in Namibia" is unchanged (still the 8 newest regardless of stock).
- Shelf order on the page: Hero → Trust banner → New in Namibia → On sale → Made in Namibia →
  From South Africa → Top rated → Explore categories → (category shelves) → Featured vendors → Newsletter.
- Sort inside a shelf — On sale: `discountPercent` desc then newest; Top rated: `averageRating` desc then
  `reviewCount` desc, only rated products with `averageRating >= 4`; origin and category shelves: newest.
- Each shelf has a "View all" link to `/discover` with the matching filter where `/discover` supports one
  (category: `?categoryId=`); otherwise plain `/discover`. *Superseded by SHS-7 (PR #129): every shelf "View all" passes `inStock=true` plus the shelf's own filter, and `/discover` reads these filters from the URL.*
- A11y/layout parity with the existing section: `<section aria-labelledby>`, `role="list"` carousel,
  the same scroll-snap behaviour on mobile, no horizontal page scroll at 360px.
- Hero: keep `fetchpriority="high"`, the `<picture>` WebP-first set and `HERO_IMAGE_SIZES` (the photo's
  rendered **width** must not change, so the LCP candidate and its `sizes` stay valid). Only the band's
  height changes.

## Hero target (SHS-1)

Current `shop.scss`: `.hero__inner` min-height 520 (base), 600 (md), 560 (xl), 600 (2xl); at xl the grid
adds `padding-block: space-10` and the photo is `aspect-ratio: 4 / 3`. Proposed ≈ **15–20% shorter**:
520→440, 600→500, 560→480, 600→520; xl `padding-block` space-10→space-8; photo aspect 4/3→3/2.
Below xl the copy block (kicker + headline + body + CTAs + the 108/148px floater shelf) can be taller than
`min-height`, so lowering `min-height` alone may change nothing on phones — the implementer measures
before/after at 390, 768, 1280, 1440, 1920 and trims the copy's vertical padding where `min-height` is not
the binding constraint. "Slightly smaller" is the brief; the numbers above are a starting point to tune by
eye against the live page.

## Out of scope

- Best sellers / trending (no sales signal), personalised or "recently viewed" shelves, a CMS/admin tool
  to curate home sections, A/B testing, new design tokens, changing the hero photo or headline copy, the mobile radial nav.
  *Superseded by SHS-6/SHS-7 (PR #129): the shelves endpoint, the `ProductQuery` params and the `/discover` filters were added. The radial nav scale was a separate follow-up (xXRUbiD5).*

## Open questions (a human should confirm — defaults chosen so work can start)

1. **Which sections?** Default: On sale, Made in Namibia, From South Africa, Top rated (+ optional category
   shelves). Drop or reorder freely — each is its own card.
2. **Top-rated threshold.** Default `averageRating >= 4` and at least 1 review (pre-launch there are few
   reviews). A single 5★ review will top the shelf; raise the minimum to 3 reviews once volume exists.
3. **Duplicates across shelves.** *Decided 2026-10-26: one appearance per page, shelves filled in page order, minimum stays 4. Done server-side in SHS-6 (PR #129).* Shipped first in PR #128 with no dedupe, so a product may appear in several shelves. *Measured on the dev box 2026-10-26:* 25 products, 23 in stock (8 Namibian, 15 South African), 5 on sale now, 16 rated 4 or higher; in-stock per category 6/5/5/5/4/2/1. Every shelf qualifies, but the page lays out about 48 card slots from 23 products, so each product shows about twice. Owner kept the page order and the minimum of 4.
4. **100-product ceiling.** *Decided 2026-10-26: one endpoint call, not seven.* Client-derivation only sees the 100 newest products, so "On sale"/"Top rated" can miss older qualifying items once the catalogue passes ~100. SHS-6 adds a single public shelves endpoint (every shelf in one response, rules applied server-side) built on new reusable `ProductQuery` filters (`onSale`, `inStock`, `originCountry`, `minRating`) and rating/discount sorts, and the web drops its client-side derivations. One round trip and one throttle hit per visit. SHS-6 also requires: the response is always the public view (so it is identical for everyone and cacheable), is cached in the API for about 60s with `Cache-Control: public, max-age=60, stale-while-revalidate` (the API JSON, not the page HTML, because of the per-request CSP nonce), and shelf items use a slim `ProductCardDto` instead of the full `ProductDto`. Not needed pre-launch. *Shipped in SHS-6 (PR #129): `GET /products/shelves`, with the 60s cache and the slim card.*
5. **Hero magnitude.** Which viewport felt too tall? Defaults above assume desktop; mobile may need a
   separate nudge.
6. **Category shelves** (SHS-5) — worth the extra page length, or is the Explore-categories tile row enough?

## Cards

| Card | Trello | Slice | Depends on | Status |
|------|--------|-------|-----------|--------|
| SHS-1 | Bn6dsEla | Shorten the `/shop` hero band | — | Merged (#127) |
| SHS-2 | lU68lYzv | Extract `ProductShelf` + shelf visibility rule (refactor, no visible change) | — | Merged (#127) |
| SHS-3 | 3tGVSIlU | "On sale" and "Top rated" shelves | SHS-2 | Merged (#128) |
| SHS-4 | mETkwXkj | "Made in Namibia" and "From South Africa" shelves | SHS-2 | Merged (#128) |
| SHS-5 | ABB1gwlR | Category shelves for the top 2 categories (optional) | SHS-2 | Merged (#128) |
| SHS-6 | vEJI4mGQ | One `/shop` shelves endpoint (all shelves in a single response, rules moved server-side) plus reusable `ProductQuery` filters and sorts, to lift the 100-product ceiling; not needed pre-launch (open question 4) | SHS-3/4/5 | Open (#129) |
| SHS-7 | 5AhR9PHK | Shelf "View all" links open a filtered `/discover` (On sale, Top rated, origin shelves are unfiltered today); `/discover` reads the SHS-6 filters and shows them as chips | SHS-6 | Open (#129) |

SHS-1 and SHS-2 can ship in parallel; SHS-3/4/5 each touch `shop.html`/`shop.ts`, so they go one at a time
(or as one `/ship-batch`) after SHS-2.

## Implementation Notes (2026-10-26) — SHS-1 + SHS-2

**Branch:** `feat/Bn6dsEla-shop-hero-height-and-product-shelf` · **PR:** #127 · one branch, sequential (SHS-2 then SHS-1).

- **SHS-2:** `app-product-shelf` (`shared/components/product-shelf/`) is presentational: inputs `shelfId`, `heading`, `subheading`, `products`, `state`, `viewAllQueryParams`, `isWishlisted`; outputs `addToCart`, `wishlistToggle`, `retry`. Copy and the 6 skeletons are inline (nothing overrides them yet). Section-header styles moved into mixins in `styles/_section.scss`, used by `shop.scss` (categories and vendors bands) and the shelf. `selectShelfProducts(products, { max = 8, min = 4 })` is the shelf rule; New in Namibia calls it with `min: 1`. The heading id is now `new-in-namibia-heading`.
- **SHS-1 finding the spec got wrong:** the hero band's height was set by its content, not `min-height`. At 1440px the headline (`display`, 68px) ran to five lines (357px) while the photo column was only ~395px, so lowering min-heights alone would do nothing. The headline now uses the `4xl` step at weight 800 (DESIGN.md updated), plus a smaller floater shelf (88px, 140px from md), less vertical padding and lower min-heights. Photo width, `<picture>` set and `HERO_IMAGE_SIZES` untouched.
- **Measured hero height (before to after):** 390px 850 to 729, 768px 670 to 606, 1280px 751 to 562, 1440px 704 to 571, 1920px 801 to 579. Desktop is 19-28% shorter, a bit past the 15-20% target.
- **Review:** `code-reviewer` FAIL on the headline using a raw `clamp` off the type scale; fixed by using the `4xl` token. It also flagged tight floater spacing at 768px (md shelf restored to 140px) and unused shelf inputs (removed).
- **Tests:** web 110 files / 1730 pass (new `product-shelf.spec.ts`); contrast gate 48 pairs pass; `npm run build` passes with `INTERNAL_API_BASE_URL` set.
- **Follow-ups:** SHS-3/4/5 are unblocked. The `4xl` headline is a visible type change a human should eyeball.

## Implementation Notes (2026-10-26) — SHS-3 + SHS-4 + SHS-5

**Branch:** `feat/3tGVSIlU-shop-home-shelves` · **PR:** #128 · one branch, sequential, one commit per card, then a review-fix commit.

- **Derivations:** `features/shop/shop-shelves.ts` holds pure, exported `deriveOnSale`, `deriveTopRated`, `deriveByOrigin(products, country)` and `deriveTopCategoryShelves(categories, products, count = 2)`. All filter to in-stock (`productStockTotal`, newly exported from `product-card.ts` and reused by the card) and go through `selectShelfProducts` (max 8, min 4). `shop.ts` exposes them as computeds behind a `shelfSource` gate that returns `[]` unless products are `loaded`; category shelves also wait for categories.
- **Page:** `shop.html` renders each shelf under an `@if (...length)` (category shelves in an `@for`), reusing `app-product-shelf` and the page's `isWishlisted` / `onAddToCart` / `onWishlistToggle`. Order: New in Namibia, On sale, Made in Namibia, From South Africa, Top rated, Explore categories, category shelves, Featured vendors.
- **Bug the review caught:** the first On sale sort used `discountPercent`, but `mappers.utils.ts` only sends it to a product's owner or an admin (`canSeeRawDiscount`), so for anonymous visitors and SSR every product scored 0 and the shelf was newest-first, and a vendor saw their own items jump to the top. The fixtures set `discountPercent`, which the public payload never has, so the tests hid it. On sale now ranks by `1 - effectivePrice / price`. **Lesson:** build web fixtures from the public payload, not from fields only the owner sees.
- **Smaller review fixes:** a View all link with filter params now has `aria-label="View all <heading>"` (the generic label stays for unfiltered links); copy no longer assumes every seller is a vendor ("Marked down for a limited time", "Namibian sellers, shipped within Namibia").
- **Copy rule held:** origin shelf copy makes no duty / tax-free / customs claim; a page test greps for it.
- **Tests:** web 111 files / 1763 pass; build passes with `INTERNAL_API_BASE_URL`. Shelves eyeballed in the browser pane against a throwaway mock API (36 products) at 1440px and 390px; real API not run.
- **Follow-ups:** open question 4 stands: shelves only see the 100 newest products; add `onSale=true` and a rating sort to `GET /products` when the catalogue nears 100. Cosmetic: `categoryShelves` sits before `topRatedProducts` in `shop.ts` (page order differs). A product can appear in several shelves (no dedupe).

## Implementation Notes (2026-10-26) — SHS-6 + SHS-7 + radial nav scale

**Branch:** `feat/vEJI4mGQ-shop-shelves-endpoint` · **PR:** #129 · one branch and PR, bundled at owner request. SHS-6 is primary, SHS-7 consumes its contract, and the radial nav scale is an independent follow-up to YS7PlgB6 (PR #121).

- **Shape:** `GET /products/shelves` (public) returns every `/shop` shelf in one response. `StorefrontShelvesService` composes `ProductsService.findShelfProducts` and `countInStockByCategory` over `createVisibleProductQuery`, `applyProductFilters` and `applyProductSort`, the same SQL as `GET /products`. Items map through `ProductToCardDto`, always the public view.
- **Rules, server-side:** in-stock only, max 8, empty below 4, `newest` min 1, top two categories, one appearance per page. One seen-set runs in page order (New in Namibia, On sale, Made in Namibia, From South Africa, Top rated, category shelves), and a dropped shelf frees its products. Category selection walks the ranked qualifying categories until two are non-empty.
- **Filters on `GET /products`:** `onSale`, `inStock`, `originCountry`, `minRating`, and `rating` / `discount` sorts. `q` with any of the four filters returns 400, because the filters are Postgres-only and search is engine-backed.
- **Cache:** 60s in-process, single-flight rebuild, per API process. `Cache-Control: public, max-age=60, stale-while-revalidate=120` is set only on success. Worst case is about 4 minutes before a changed discount or stock shows for a returning browser. The cart re-prices server-side, so the charge is correct.
- **Categories:** `GET /categories` carries `CategoryDto.productCount`. No schema change, no migration.
- **SHS-7 URL rules on `/discover`:** `onSale` and `inStock` accept the literal `true` only. `originCountry` accepts upper-case `NA` or `ZA` only. `minRating` is a finite number from 1 to 5. A `q` wins and drops the filters, including from the header search. Rating and discount sorts are URL-only, so the sort dropdown shows a disabled label for the selected sort.
- **Shelf vs View all:** with page-wide dedupe a shelf is a subset of its View all list (same filter and sort), not necessarily its head. This supersedes the "first items match the shelf" wording in SHS-7.
- **Payload:** 508 seeded products, two images each. Shelves response 29,359 B for 36 cards (~815 B each), against 179,344 B for `GET /products?limit=100` (~1,793 B each). About 84% smaller. The bulk rows are platform listings, so the vendor block is lightly covered.
- **Bugs caught by the live check and review:** `ProductsService.list()` on the web did not serialise the four filters, so a filtered `/discover` showed chips over an unfiltered list; the specs only asserted the query object. Review found `Cache-Control: public` on a 500, and a rating or discount sort surviving a search and returning when the search was cleared. All fixed.
- **Small catalogues:** on the 11-product dev DB only New in Namibia shows. That is dedupe plus the minimum of 4, as decided.
- **Radial nav:** the card's "600px tall scales to ~0.87" is a miscalculation, since the height term only bites below ~314px. The design's home item already overlaps the 92px FAB, so the clamp only stops it getting worse. At 320px every chip is on-screen, but "Wishlist" and "Theme" ellipsize because the rings must nest. Full labels fit from ~360px, and 375px was checked in the browser. Commit `fc6b04c` overstates the 320px result.
- **Tests:** api 1905 pass, 14 skipped (the opt-in DB specs, `HB_DB_TESTS=1`, pass when run locally). web 111 files, 1781 pass. Lint and build clean.
- **Lessons:** a spec that asserts only the query object misses a field dropped at the request level, so check the request. Build fixtures from the public payload (the On sale bug in #128 came from owner-only fields). The cache is per process, so each replica rebuilds once per TTL.
- **Follow-ups:** decide the 320px label treatment with design (shorter labels, or a different layout below 360px); wire the DB specs into a DB-backed CI job; owner to confirm the ~4 minute staleness window or lower SWR; per-replica cache if the API scales out; index only if the catalogue reaches thousands.
