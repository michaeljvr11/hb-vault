# Session 9 — 2026-07-02

## Cards worked
b4VoyjRu (API discovery filters) + 6qlkwk75 (storefront browse+search UI) + board management.
Scope expansion + one critical review finding + follow-up card creation.

## What changed
Full-stack product discovery: API category/vendor/search filters, unified suggest endpoint, and storefront
search+browse UI with product-detail routes. All shipped together on branch `fable-storefront-oneshot`.

Key changes:
- `libs/shared/src/contracts/product.ts` — `ProductQuery` interface + search suggest response types
- `apps/api/src/search/` — new module: `GET /search/suggest` (unified omnibox suggestions), `ProductQueryDto` validator
- `apps/api/src/products/products.service.ts` — `findAll` extended with categoryId/q/vendorId filters + approvedVendorOnly composition
- `apps/web/src/app/shared/components/` — product-card grid/carousel, category-chips, search-bar omnibox w/ grouped suggestions + keyboard nav + ARIA, trust-banner, vendor-showcase, radial-nav (Angular port of mobile design)
- `apps/web/src/app/pages/shop/` — rebuilt home to desktop + NEW mobile designs, mobile sticky search + chips deep-link to /discover
- `apps/web/src/app/pages/discover/` — new SSR route for category/vendor/search filters (URL-driven q/categoryId/vendorId state, dismissible vendor chip, SME client-side toggle, distinct empty state)
- `apps/web/src/app/pages/products/` — new SSR route /products/:id (gallery, shipping card, vendor card w/ View Storefront→/discover?vendorId=, related products, sticky add-to-cart with anon→login returnUrl gate, 404 state)
- 175 API + 395 Web unit tests, all passing

## Key decisions
- **Scope expansion:** vendorId param + unified suggest endpoint added beyond spec v1 per owner requirement ("vendor-specific marketplace → users search vendors/categories/products")
- **Vendor suggestions → /discover?vendorId=:** no vendor-profile page/design exists yet; product-detail shows single-currency price (ProductDto has one price+currency; NAD parity display deferred)
- **Approved-vendor visibility compose:** discovery filters apply to platform listings only via TypeORM Brackets (fix #2 below)
- **Client-side SME toggle:** filters where product.vendor presence (no API gate)
- **Pagination/sort/currency parity deferred:** created fast-follow cards (#44 rmpsJVLi, #40 UG5UFyxy for vendor-profile)

## Review & fix process
Code review verdict FIX-FIRST (2 FAILs):
1. **FALSE-GREEN SPECS:** `ProductsService.findAll` approved-vendor filter was never exercised; tests used fake QueryBuilder lacking SQL AND/OR semantics. Added regression guards + precedence-faithful fake QueryBuilder. All green now.
2. **PRECEDENCE BUG (critical):** unparenthesized OR in SQL let platform listings bypass ALL filters (OR binds weaker than AND). Wrapped visibility clause in TypeORM Brackets. Guards added that fail on pre-fix code.

Two WARNs (both addressed): SME badge color (#e8f5e9) confirmed in design exports; single queryParamMap source via toSignal+effect in discover.

Final gate: test:api 175/175 (12 suites), web 395/395 (37 files), lint:api clean, npm run build clean.

## Test/review outcome
- `npm run test:api` → 175 passed
- `npm run test -w @hb/web` → 395 passed
- `npm run lint:api` → clean
- `npm run build` → clean
- Code review: FIX-FIRST → SHIP (2 FAILs + 2 WARNs all addressed)

## PR
#26 (https://github.com/michaeljvr11/hb-mono-repo/pull/26) — open, awaiting human merge.

## Follow-ups
- **Card #40 UG5UFyxy (vendor profile page)** — vendor suggestions currently land on /discover?vendorId=; profile page with stats/reviews is a follow-up
- **Card #44 rmpsJVLi (pagination + sort)** — discovery results are unbounded; pagination + sort skipped to ship faster
- **DesignSync push:** the design tokens/screenshots exported to claude.ai need `/design-login` (interactive terminal only). Deferred to Josh.
