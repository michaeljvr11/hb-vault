# Product Wishlist

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Auth & Roles]] · [[Public Storefront & SSR]] · [[Customer Profile]] · [[Money & Currency Rules]]

## Problem

Customers have no way to save a product for later. The only wishlist artifacts in the
codebase are two dead stubs that predate any backend:

- `apps/web/src/app/shared/components/radial-nav/radial-nav.ts` carries
  `{ id: 'wishlist', icon: 'favorite', label: 'Wishlist' }` with **no `routerLink`** — unlike
  `home`/`search`, which navigate directly.
- `apps/web/src/app/features/shop/shop.ts` `onRadialNavSelect()` catches that emit and shows a
  **"Wishlist is coming soon"** snackbar.
- The desktop nav bar (`apps/web/src/app/layout/nav-bar/`) has no wishlist affordance at all —
  cart and the account icon are the only icons.

There is no `Wishlist` entity in [[HB Domain Model]], no table, no endpoint, no contract in
`libs/shared`. This is net-new, confirmed against `git log` and a branch search.

Goal: a signed-in customer can toggle a product into/out of a personal wishlist from any
product card and from the product-detail page, see a badge count in both nav surfaces, and
view the saved products in a list view.

## Scope

**Backend** (`apps/api`)
1. `wishlist_items` table + TypeORM migration. Single table: `id`, `userId` (FK users,
   CASCADE), `productId` (FK products, CASCADE — mirrors `cart_items`), `createdAt`, plus a
   `UNIQUE (userId, productId)` constraint.
2. `apps/api/src/wishlist/` module mirroring the `cart/` file layout: entity, controller,
   service, module, `dto/add-wishlist-item.dto.ts`.
3. Endpoints, all authenticated (global `JwtAuthGuard`, no `@Public`, no `@Roles` — any
   signed-in role, same as cart):
   - `GET /wishlist` → `WishlistDto`
   - `POST /wishlist/items` → `WishlistDto` (idempotent)
   - `DELETE /wishlist/items/:productId` → `WishlistDto`

**Frontend** (`apps/web`)
4. `core/api/wishlist.service.ts` — signal-backed client modelled on `CartService`, with an
   `itemCount` computed and an O(1) `has(productId)` membership lookup.
5. Heart toggle on the shared `ProductCard`, wired into all four consumer templates
   (shop, discover, public vendor profile, PDP related-products grid).
6. "Save to wishlist" action in the PDP hero (not an `app-product-card`, so it needs its own).
7. `/wishlist` list view — image, name, live price, remove, add-to-cart; loading/loaded/
   empty/error states.
8. Nav: desktop nav-bar icon + badge (cart-icon pattern), and the radial-nav stub gets
   `routerLink: '/wishlist'` + a badge; the coming-soon branch in `shop.ts` is deleted.

## Business rules it must honour

From [[Auth & Roles]] and [[Public Storefront & SSR]] — enforce, do not reinvent:

- **Auth is settled.** JWT access + rotating httpOnly refresh cookie, global guards,
  `@Public`/`@Roles`. Extend it; never fork it.
- **Ownership checks live in the service layer.** Every query is scoped by `userId`; touching
  a row that isn't the caller's is a plain **404, never 403** — the `OrdersService.findOneForUser`
  / `CartService.findOwnedItem` pattern. No existence leak.
- **Browse is public, transact is authenticated.** A wishlist is account-bound, so it sits on
  the transact side of the boundary alongside add-to-cart. An anonymous heart click routes to
  `/login?returnUrl=<current url>` (open-redirect-safe via `sanitizeReturnUrl`) — it must not
  silently no-op and must not surface a 401.
- **Money is never snapshotted here.** Per [[Money & Currency Rules]], price snapshots happen
  at order creation only. Wishlist rows store a product reference and nothing else; price,
  currency, stock and image are resolved **live at read time**, exactly like `CartService.toDto`.
  ZAR and NAD are never conflated — display uses the shared `format-price` util with each
  item's own currency. There is no wishlist "total" of any kind.
- **Schema change ⇒ migration.** `synchronize` stays off. New table, so no backfill/DEFAULT
  concern, but `down()` must drop cleanly.
- **Every endpoint input ⇒ a class-validator DTO implementing the shared interface.**
- **SSR safety is non-negotiable.** All four product-card host pages are `RenderMode.Server`
  and render anonymous on the server. Wishlist membership and the nav badge must be gated
  behind the `hydrated` signal + `afterNextRender` pattern already in `nav-bar.ts`, or the
  first client render diverges from the server DOM.

## Shape decisions taken (flag if wrong — do not silently change)

- **Single `wishlist_items` table**, NOT the cart's two-table `Cart` + `CartItem` shape. A
  wishlist is a pure set of (user, product): no container identity, no `getOrCreate`, no
  totals, no checkout semantics, no `updatedAt` meaning. The cart's container exists because
  the cart is a staging area with a lifecycle; a wishlist has none.
- **Delete by `productId`, not by row id** (`DELETE /wishlist/items/:productId`), diverging
  from `DELETE /cart/items/:id`. A heart toggle on a product card knows the productId and
  never the wishlist row id; delete-by-row-id would force a client-side lookup on every toggle.
- **`POST` is idempotent** — re-adding an already-wishlisted product returns 200 with the
  unchanged list, not 409. A double-tapped heart must not produce an error toast.
- **`POST` does not reject out-of-stock products**, unlike `CartService.addItem` which throws
  `ConflictException`. Saving something for later is exactly what you do when it's out of stock.
- **`ProductCard` stays presentational.** It is `input`/`output` only today; membership arrives
  as a `wishlisted` input and the toggle leaves as a `wishlistToggle` output, rather than
  injecting `WishlistService` into the component. Cost: a one-line binding change in each of
  the four host templates. Benefit: the component stays testable and consistent with `addToCart`.

## `@hb/shared` contract impact

New `libs/shared/src/contracts/wishlist.ts` (interfaces only, exported from `contracts/index.ts`):

- `WishlistItemDto { id; productId; productName; price; currency: CurrencyCode; stockQuantity; imageUrl?; addedAt: string }`
- `WishlistDto { items: WishlistItemDto[]; itemCount: number }`
- `AddWishlistItemRequest { productId: string }`

Follows the `contracts/cart.ts` "live-resolved product display fields, never snapshotted"
convention, with three deliberate differences: **no `quantity`, no `lineTotal`, no `totals`**.
`itemCount` is a row count (there are no units to sum). Field named `price` rather than the
cart's `unitPrice`, mirroring `ProductDto.price`, since there is no per-unit concept.

No changes to `CartDto`, `ProductDto`, or `AnalyticsEventType`.

## Out of scope

- **Wishlist analytics.** `libs/shared/src/enums/analytics-event-type.ts` documents its list as
  "intentionally final … without a fresh review", so no `ADD_TO_WISHLIST` event in v1. Follow-up card.
- **Intent replay across login.** An anonymous heart click sends the user to `/login?returnUrl=…`
  and returns them to the page, but does **not** auto-wishlist the product afterwards. This is
  parity with the existing add-to-cart gate, which behaves the same way today.
- **Anonymous/guest wishlists** (localStorage or session-backed) and merge-on-login.
- **Sharing / public wishlists, multiple named lists, notes or priority per item.**
- **"Move all to cart", sorting, filtering, pagination** of the wishlist.
- **Back-in-stock or price-drop notifications** — no email/notification provider is wired
  beyond the existing verify/reset token flow.
- **A "no longer available" tombstone row.** `Product` has no soft-delete or publish flag
  (no `isActive`/`status` column), so deleted products vanish via the FK cascade. Adding a
  tombstone means adding product soft-delete first — a separate card. See open question 3.
- **Rendering `<app-radial-nav>` beyond `/shop`.** It is only used in `shop.html` today, so
  mobile wishlist nav reach is the storefront home only. Not expanded here.

## Open questions (confirm before /ship-card)

1. **Standalone `/wishlist` route, or a fourth [[Customer Profile]] `ProfileShell` tab?**
   Recommendation: **standalone**. The radial-nav stub already treats `wishlist` as a sibling
   of `profile`, and a wishlist is a shopping surface, not account admin. WL-4 is written for
   the standalone option — if the answer flips, its route ACs need rewriting before it starts.
2. **Duplicate prevention.** Recommendation: **yes** — a DB-level `UNIQUE (userId, productId)`,
   with `POST` idempotent rather than 409. Confirm the idempotent-vs-409 half specifically, as
   it changes the client's error handling on a rapid double-tap.
3. **Unavailable wishlisted products.** Recommendation for v1: out-of-stock items **stay** in
   the list with an "Out of stock" badge and a disabled add-to-cart (`stockQuantity` is already
   resolved live, so this costs nothing); deleted products disappear silently via the FK
   cascade; **no** tombstone row. Confirm, or raise a `Product` soft-delete card first.
4. **Wishlist badge in the desktop nav — icon-only, or icon + count?** Recommendation: icon +
   count, matching `nav-bar__cart-badge`, so the two icons read consistently.

## Vertical slices → Trello cards

| # | Title | Card ID |
|---|---|---|
| WL-1 | Wishlist data model + API — `@hb/shared` contract, migration, GET/POST/DELETE `/wishlist` | rtBV85cQ |
| WL-2 | `WishlistService` (web) + heart toggle on `ProductCard` across storefront surfaces | MiiSqOS5 |
| WL-3 | "Save to wishlist" action on the product-detail page (PDP hero) | RrgIdrIo |
| WL-4 | `/wishlist` list-view page + route (authGuard, client-rendered) | zG9zNAh8 |
| WL-5 | Wishlist nav entry + badge on desktop nav-bar and mobile radial-nav | L54YO6fI |

Order: WL-1 → WL-2 → {WL-3, WL-4 in parallel} → WL-5.

**Known conflict:** WL-5 touches `radial-nav.ts`'s items array and `shop.ts`'s `labels` map —
the same lines as the in-flight PR wiring the radial-nav `profile` item to `routerLink="/profile"`.
Rebase on that PR; do not revert its change.
