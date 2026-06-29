# Listing Types & Vendor Rules

Two business models in one schema (`ListingType` on products and order items):

1. **`platform`** — first-party: HB sources the product and arranges cross-border fulfilment. **No vendor attached.**
2. **`vendor`** — marketplace: a third-party vendor owns the listing and fulfils the order.

## Hard invariant (DB CHECK constraint)

`products.vendorId` is nullable with a CHECK: **vendor listings must have a vendor; platform listings must not.** There is deliberately no fake "house vendor" row — that would pollute vendor reporting and onboarding.

- Admins creating products → platform listings.
- Vendors creating products → vendor listings (enforced in `ProductsService.createWithImages`).
- An order can mix both models; `order_items` snapshots `listingType` + `vendorId` per line.

## Vendor lifecycle (`libs/shared/src/enums/vendor-status.ts`)

```
pending → approved
        ↘ rejected
approved → suspended (and back, by admin)
```

Only `approved` vendors can list products and receive orders (business rule — enforce in service layer).

## V1 onboarding depth (confirmed 2026-06-18)

**Document upload is NOT required for v1.** The existing `CreateVendorRequest` fields
(`businessName`, `tradingName?`, `registrationNumber?`, `website?`, `description?`,
`countryCode?`) are sufficient to create a `pending` vendor. `verificationDocumentUrl`
on the `Vendor` entity remains optional and can be populated later.

KYC / banking / verification-document review is a **future card** — leave the
`verificationDocumentUrl` field in place but don't build upload/review UI in v1.

## Rules for agents

- Never bypass the vendorId CHECK semantics in code paths that create/update products.
- Vendor-facing queries must filter by ownership — a vendor sees only their own listings/orders.
- Vendor status changes are admin-only actions (see [[Auth & Roles]]).
- Vendor order/fulfilment transitions are limited — see [[Order State Machine]] for the
  confirmed transition rights (vendor: `confirmed→processing`, `processing→handed_to_hb` only).

## TBD (still open)

- Commission/fee structure per vendor sale.
- Whether vendors can fulfil cross-border themselves or must use platform logistics.
- KYC depth / banking details for vendor payout (deferred past v1).

Related: [[HB Domain Model]] · [[Money & Currency Rules]]


## Implementation Notes — Public vendor directory (2026-06-18)

**Card L9fM1Gu7 "Implement Store front"** shipped via PR #4. New public `GET /vendors/directory` endpoint returns only `approved` vendors (enforcing the rule that only approved vendors are public-facing). The storefront at `/shop` consumes this to render "Featured SME Vendors" alongside a hero, product carousel, trust banner, category grid, and newsletter signup.

**Key decisions:** the endpoint has no pagination/filtering (cold-start MVP — can upgrade when storefront grows); vendor dto doesn't include logo/rating/city (model contract hasn't been extended; those visual elements from the Claude Design export are either omitted or derived client-side). The storefront's seed script (`npm run seed`) is all-or-nothing on category count — consider per-entity idempotency in a later refactor.

**Related:** [[HB Domain Model]] · PR https://github.com/michaeljvr11/hb-mono-repo/pull/4

### Approved-only filter extended to product catalogue (2026-06-29, card #36 c2o6xfZs)

**Card #36 "Filter vendor listings by approved status on public product endpoints"** shipped via PR #23. The approved-only visibility rule for vendors (see slice above) is now extended to the public product catalogue. This gates the discovery and detail endpoints `GET /products` and `GET /products/:id` to ensure non-approved vendors' products are not exposed.

**What shipped:**
- `apps/api/src/products/products.service.ts` — `findAll` and `findOne` now filter public product results via a FindOptions `where` array-of-conditions (OR-of-AND): return products where `listingType = 'platform'` OR (`listingType = 'vendor'` AND `vendor.status = 'approved'`). Platform (first-party) listings always visible. Non-approved vendors' products are absent from `GET /products` and 404 on `GET /products/:id` (hits existing `NotFoundException`) — even when requested by direct id. Imports `VendorStatus` from `@hb/shared` (already exported).
- `apps/api/src/products/products.service.spec.ts` — NEW, 14 unit tests covering both endpoints: `findAll` (platform always visible, approved vendors visible, pending/rejected/suspended excluded via real WHERE-array mock, exact where/relations assertion, empty repo, mixed fixture); `findOne` (platform + approved resolve, each non-approved status → 404 by direct id, unknown id → 404).

**Key decisions:**
- **Applied the same approved-only rule already shipped for vendors** (`GET /vendors/directory`, `GET /vendors/:id` in card vs3y5tDJ / PR #21) to maintain consistent public-facing logic.
- **Service/query-layer filter, no controller change.** Reuses existing visibility pattern; no new endpoint created.
- **Chose FindOptions array-of-conditions over QueryBuilder** for minimal diff. The `vendor` relation is ManyToOne so the nested-relation join is safe and does not truncate the eager `images`/ManyToMany `categories` collections.
- **No `@hb/shared` change, no migration, no schema change.** All fields already existed; this is a visibility + filtering change only.

**Tests & build:**
- `npm run test:api` → 126 passed (+14 new in products.service.spec.ts).
- `npm run test -w @hb/web` → 293 passed (no web files touched).
- `npm run lint:api` → clean.
- `npm run build` → clean (only pre-existing `shop.scss`/`admin-catalog.scss` budget warnings, unrelated).

**Code review outcome:**
- SHIP. No FAILs. One forward-looking follow-up (out of scope for #36, relevant to card #37 "Category filter + search on GET /products"): if `findAll` ever gains pagination, this FindOptions shape with collection joins would truncate `images`/`categories` and must move to a QueryBuilder with `relationLoadStrategy: 'query'` or distinct pagination. Card #37 also edits `findAll` and must AND its predicates with this approved-vendor filter so no non-approved listing leaks via any param.

**PR:** #23 (https://github.com/michaeljvr11/hb-mono-repo/pull/23) — open, awaiting human merge.

**Related:** [[Public Storefront & SSR]]
