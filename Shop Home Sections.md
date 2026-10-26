---
type: spec
tags:
  - storefront
  - shop-home
  - ui
---

# Shop Home Sections

Status: **SHS-1 and SHS-2 shipped 2026-10-26** ([PR #127](https://github.com/michaeljvr11/hb-mono-repo/pull/127), awaiting human merge); SHS-3/4/5 still in To Do.
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
  new shelf is **already in memory** — no new request, no new endpoint.
- `ProductDto` already carries everything these shelves need, server-computed: `isDiscountActive`,
  `effectivePrice`, `discountPercent`, `averageRating`, `reviewCount`, `originCountry`, `listingType`,
  `categories`, `stockQuantity`, `sizes`. **`@hb/shared` impact: none.**
- `ProductQuery` supports only `categoryId`, `q`, `vendorId`, `page`, `limit`, `sort` (`relevance | newest |
  price_asc | price_desc | name`). There is no `onSale` filter and no rating sort. Adding them is the
  scale-up path (see Open questions), not needed today.
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

- All new shelves are **derived client-side from the already-loaded product list** by pure, exported,
  unit-tested functions (same pattern as `deriveCategoryCounts`).
- **Max 8 products per shelf; a shelf renders only with at least 4 products.** Below that the whole
  section is omitted (no empty state, no heading) — a half-empty shelf looks broken.
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
  (category: `?categoryId=`); otherwise plain `/discover`. No new `/discover` filters are added here.
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
  to curate home sections, A/B testing, new API endpoints, new `ProductQuery` params, changes to
  `/discover`, new design tokens, changing the hero photo or headline copy, the mobile radial nav.

## Open questions (a human should confirm — defaults chosen so work can start)

1. **Which sections?** Default: On sale, Made in Namibia, From South Africa, Top rated (+ optional category
   shelves). Drop or reorder freely — each is its own card.
2. **Top-rated threshold.** Default `averageRating >= 4` and at least 1 review (pre-launch there are few
   reviews). A single 5★ review will top the shelf; raise the minimum to 3 reviews once volume exists.
3. **Duplicates across shelves.** Default: no dedupe (a product may appear in New in Namibia and On sale).
   With a tiny catalogue the shelves will overlap heavily; the ≥4 rule hides the thin ones.
4. **100-product ceiling.** Client-derivation only sees the 100 newest products, so "On sale"/"Top rated"
   can miss older qualifying items once the catalogue passes ~100. Follow-up when that happens: add
   `onSale=true` and a rating sort to `ProductQuery` (shared contract + API + migration-free), then each
   shelf fetches its own `limit=8`. Not needed pre-launch.
5. **Hero magnitude.** Which viewport felt too tall? Defaults above assume desktop; mobile may need a
   separate nudge.
6. **Category shelves** (SHS-5) — worth the extra page length, or is the Explore-categories tile row enough?

## Cards

| Card | Trello | Slice | Depends on |
|------|--------|-------|-----------|
| SHS-1 | Bn6dsEla | Shorten the `/shop` hero band | — |
| SHS-2 | lU68lYzv | Extract `ProductShelf` + shelf visibility rule (refactor, no visible change) | — |
| SHS-3 | 3tGVSIlU | "On sale" and "Top rated" shelves | SHS-2 |
| SHS-4 | mETkwXkj | "Made in Namibia" and "From South Africa" shelves | SHS-2 |
| SHS-5 | ABB1gwlR | Category shelves for the top 2 categories (optional) | SHS-2 |

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
