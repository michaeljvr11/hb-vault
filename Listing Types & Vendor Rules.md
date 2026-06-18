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

## Rules for agents

- Never bypass the vendorId CHECK semantics in code paths that create/update products.
- Vendor-facing queries must filter by ownership — a vendor sees only their own listings/orders.
- Vendor status changes are admin-only actions (see [[Auth & Roles]]).

## TBD (ask a human)

- Vendor onboarding requirements (KYC, banking details).
- Commission/fee structure per vendor sale.
- Whether vendors can fulfil cross-border themselves or must use platform logistics.

Related: [[HB Domain Model]] · [[Money & Currency Rules]]


## Implementation Notes — Public vendor directory (2026-06-18)

**Card L9fM1Gu7 "Implement Store front"** shipped via PR #4. New public `GET /vendors/directory` endpoint returns only `approved` vendors (enforcing the rule that only approved vendors are public-facing). The storefront at `/shop` consumes this to render "Featured SME Vendors" alongside a hero, product carousel, trust banner, category grid, and newsletter signup.

**Key decisions:** the endpoint has no pagination/filtering (cold-start MVP — can upgrade when storefront grows); vendor dto doesn't include logo/rating/city (model contract hasn't been extended; those visual elements from the Claude Design export are either omitted or derived client-side). The storefront's seed script (`npm run seed`) is all-or-nothing on category count — consider per-entity idempotency in a later refactor.

**Related:** [[HB Domain Model]] · PR https://github.com/michaeljvr11/hb-mono-repo/pull/4
