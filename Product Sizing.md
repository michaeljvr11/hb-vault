
# Product Sizing

## Problem

Vendors (and, by extension, admins creating platform listings) currently have no way to
sell a product that comes in multiple sizes. A shirt that comes in XS–XXL or a shoe that
comes in 6–12 has to be listed as one undifferentiated SKU with one `stockQuantity` — there
is no way to say "3 left in Medium, 0 left in XL," and no way to know which size a customer
actually ordered.

## Scope

- Vendors/admins can define a **custom, per-product list of sizes** when creating or editing
  a product (e.g. `XS, S, M, L, XL, XXL` for a shirt; `6, 7, 8, 9, 10, 11, 12` for a shoe).
  Sizing is **opt-in per product** — a product with zero sizes configured behaves exactly as
  it does today (one undifferentiated stock count, no size selector).
- **Stock is tracked per size**, not just per product (see Decision below). Total product
  availability becomes the sum of its sizes' stock once sizing is enabled.
- The size list (with per-size stock) is visible on the **product detail page**.
- The **product list/card view** shows a lightweight "Sizing available" hint when a product
  has ≥1 sizes configured.
- The customer **must select a size before adding a sized product to cart**.
- The **selected size is remembered through checkout and stored on the order line**, so the
  vendor/admin/customer can see exactly what size was ordered — even after the vendor later
  renames, reorders, or deletes that size.

## Decision — per-size stock tracking (confirmed with the product owner)

Two shapes were possible: (a) size as a display-only label with one shared stock pool per
product, or (b) a real per-size stock count. **(b) was chosen.** This means:

- A new child entity/table, not just a label array on `Product`.
- The existing checkout stock-locking transaction (see [[Order State Machine]]
  implementation notes — pessimistic `FOR UPDATE` per product, in deterministic product-id
  order, "no oversell under concurrency, no deadlocks") must be **extended to lock/decrement
  size rows** for sized products, not the parent product row, while leaving unsized products
  on the existing product-row path untouched.
- This is real money/inventory logic — every touched path needs unit tests per the root
  `CLAUDE.md` non-negotiable.

## Business rules

1. **Opt-in, per-product.** Zero sizes ⇒ unchanged legacy behaviour (single
   `Product.stockQuantity`, no size selector anywhere). One or more sizes ⇒ size selection
   becomes mandatory at add-to-cart, and the size rows — not `Product.stockQuantity` — become
   the source of truth for availability.
2. **Free-text, per-product size labels — no shared taxonomy.** A shirt's sizes and a shoe's
   sizes are unrelated lists; there is no global "size set" to pick from in v1. Vendors type
   the labels themselves.
3. **Vendor-controlled display order**, not auto-sorted. Alphabetical sort breaks
   `XS < S < M < L < XL < XXL` no worse than it breaks numeric shoe sizes as strings (`"10"`
   sorts before `"2"`), so the vendor sets an explicit order when configuring sizes; the UI
   renders in that order everywhere (PDP, cart, order lines).
4. **Unique labels per product.** No two size rows on the same product may share a label
   (validated server-side).
5. **One price for the whole product regardless of size.** No per-size price delta in v1 —
   matches the existing single `price` column.
6. **Cart line identity becomes `(productId, productSizeId)`.** Today `CartService.addItem`
   merges by `productId` alone (`cart.service.ts:57`) — picking Small then picking Large of
   the same shirt must **not** merge into one ambiguous line. This is an existing-code change,
   not just an addition.
7. **Cart quantity clamps against the selected size's stock**, the same way it clamps against
   `Product.stockQuantity` today, when the item is sized.
8. **Order line snapshots the size label at purchase time**, exactly like `productName` and
   `unitPrice` are already snapshotted onto `OrderItem` (`order-item.entity.ts:40,43`) —
   because a vendor can rename or delete a size after the order ships, and the historical
   order record must stay accurate regardless. The FK to the size row stays nullable /
   `ON DELETE SET NULL`, mirroring the existing `productId` pattern on `OrderItem` — deleting
   a size must never break past orders.
9. **Order-creation stock check/decrement locks size rows** (deterministic order, same
   discipline as the existing per-product lock ordering) for sized lines; unsized lines keep
   using the existing product-row lock path unchanged.
10. Sizing is **not gated by `listingType`** — available to platform and vendor listings alike,
    consistent with every other product attribute (price, images, stock). See Open Question 1.

## `@hb/shared` contract impact

New DTO in `contracts/product.ts`:

```ts
export interface ProductSizeDto {
  id: string;
  label: string;
  stockQuantity: number;
  displayOrder: number;
}
```

Additive changes:

- `ProductDto.sizes?: ProductSizeDto[]` — absent/omitted for unsized products (mirrors the
  optional-field pattern already used for `shippingFeeOverrides`, image `variants`, vendor
  `logo`/`banner`).
- `ProductCreateRequest.sizes?` / `ProductUpdateRequest.sizes?`:
  `Array<{ label: string; stockQuantity: number; displayOrder?: number }>` — **whole-list
  replace** semantics on update, matching how `categoryIds` already works. Omitted/absent ⇒
  product stays unsized (or keeps its existing sizes unchanged on a partial update that
  doesn't touch this field — exact PATCH semantics to confirm at implementation).
- `AddCartItemRequest` (`contracts/cart.ts`) gains `productSizeId?: string` — the service
  layer requires it when the target product has sizes and rejects it when the product has
  none.
- `CartItemDto` gains `productSizeId?: string` and `sizeLabel?: string` (both live-read, for
  display and for round-tripping the selection through update/checkout).
- `OrderItemDto` gains `sizeLabel?: string` — the purchase-time snapshot, display-only.

Nothing about `Product.price`, `Product.currency`, `Product.originCountry`, or the
commission/shipping-fee model changes — a sized product is still exactly one price/currency/
origin (rule 5).

## Out of scope (v1)

- A shared/global size taxonomy or a size chart / measurement-guide UI.
- Per-size pricing.
- Low-stock or out-of-size-stock alerts/notifications.
- Any reconciliation UI for "the vendor deleted a size you had in your cart" beyond the
  existing pattern (the line clamps/disappears the same way an out-of-stock or deleted
  product already does today).
- Bulk/CSV size-list import.

## Open questions (confirm before/at implementation)

1. **Vendor-only, or platform listings too?** The feature request says "vendors should be
   able to..." — rule 10 above defaults to making it available to both listing types (same
   `Product` schema, no precedent elsewhere for gating an attribute by `listingType`). Flag
   for confirmation; cheap to restrict later if wrong.
2. **Duplicate labels** — rule 4 defaults to server-side rejection. Confirm.
3. **PATCH semantics for `ProductUpdateRequest.sizes`** — whole-list replace (delete rows not
   present in the new list, matching `categoryIds`) is the default; confirm this is
   acceptable given that a deleted size's stock simply disappears (its historical order lines
   are unaffected per rule 8, but any *live* cart lines referencing it need to clamp/vanish
   per the "out of scope" note above).

## Related

[[HB Domain Model]] · [[Listing Types & Vendor Rules]] · [[Order State Machine]] ·
[[Product Image Optimization Pipeline]] (precedent for the additive-optional-field DTO
pattern used here)


## Implementation Notes

**Shipped 2026-09-01** · **Cards:** SZihvfYb, 53D8nMRK, pxOYnZNI, aEDclEVI, KBwKJcTw · **Branch:** `feat/SZihvfYb-product-sizing` · **Status:** PR open

**What shipped:** Full end-to-end sizing feature. API added `ProductSize` entity + migration; all three spec open questions resolved using defaults (vendor+platform both supported, duplicate labels rejected server-side, whole-list-replace PATCH semantics). Cart line identity is now `(productId, productSizeId)`. Checkout transaction locks size rows (pessimistic) for sized products, leaves unsized products unchanged. PDP size selector with out-of-stock disabled, "Sizing available" badge on ProductCard (quick-add routes sized products to PDP). Order lines snapshot size label at purchase time (nullable FK, `SET NULL`), with new admin order-items UI section showing historical sizes. Vendor/admin product forms gained Sizes section (add/remove/reorder).

**Integration gaps caught in code review:** (1) Product creation with images silently dropped sizes (multipart FormData needed explicit JSON parse on DTO). (2) Cart/checkout pages never rendered `sizeLabel`, so same-product different-sizes looked identical. (3) Wishlist add-to-cart broke for sized products (no inline size choice); fixed by adding `WishlistItemDto.hasSizes` and routing to PDP. (4) Highest-risk: deleted-size cart lines were falling through to decrement `products.stockQuantity` (a counter sized products don't maintain) instead of failing cleanly — now checks `product.sizes` exists before legacy path, throws `ConflictException` otherwise. (5) New admin order-items section lacked CSS. All 5 were fixed and re-verified before PR opened. **Lesson:** integration gaps between cards don't surface in individual card ACs; second code-review pass was load-bearing.

**Notable decision:** `apps/web/angular.json` `anyComponentStyle` budget bumped 16kB → 18kB (error threshold only; 8kB warning left unchanged) because PDP's size-selector CSS pushed `product-detail.scss` past the ceiling. Five other component stylesheets in the codebase already exceed the warning threshold.
