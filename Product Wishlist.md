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
- **Rendering `<app-radial-nav>` beyond `/shop`.** ~~It is only used in `shop.html` today, so
  mobile wishlist nav reach is the storefront home only.~~ **Correction (2026-08-10, found
  during WL-5 implementation): this claim was wrong.** `<app-radial-nav>` is rendered on four
  templates — shop, discover, product-detail, and vendor-profile — all already binding
  `[cartCount]`. See Implementation Notes below. Radial-nav's page *reach* (which pages render
  it) is unchanged by this batch — still not added to new pages. Not expanded here.

## Open questions (confirm before /ship-card)

1. ✓ **Standalone `/wishlist` route, or a fourth [[Customer Profile]] `ProfileShell` tab?**
   **Resolved 2026-08-06, confirmed with owner: standalone.** WL-4 is written for this;
   no rewrite needed.
2. ✓ **Duplicate prevention.** **Resolved 2026-08-06, confirmed with owner: idempotent
   `POST`** (200, unchanged list on re-add) — not a 409. DB-level `UNIQUE (userId, productId)`
   stands as the enforcement mechanism.
3. ✓ **Unavailable wishlisted products.** **Resolved 2026-08-06, confirmed with owner:**
   out-of-stock items stay in the list with an "Out of stock" badge and a disabled
   add-to-cart. Deleted products disappear silently via the FK cascade; no tombstone row.
4. ✓ **Wishlist badge in the desktop nav — icon-only, or icon + count?**
   **Resolved 2026-08-06, confirmed with owner: icon-only** — no count badge on the
   desktop `nav-bar` (diverges from `nav-bar__cart-badge`, deliberately). The mobile
   radial-nav wishlist item keeps its count badge (matching its cart item) — this
   question was scoped to desktop only.

All open questions resolved — this spec is ready for `/ship-card`.

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

## Implementation Notes (2026-08-10)

Shipped as five slices on one branch, `feat/rtBV85cQ-product-wishlist`, one PR for the whole
batch: [hb-mono-repo#49](https://github.com/michaeljvr11/hb-mono-repo/pull/49).

**WL-1 — API + contract** (`rtBV85cQ`): `libs/shared/src/contracts/wishlist.ts` —
`WishlistItemDto`/`WishlistDto`/`AddWishlistItemRequest`, no `quantity`/`lineTotal`/`totals`;
`itemCount` is a row count; field is `price`, not the cart's `unitPrice`. Migration
`1784678400000-WishlistItems.ts` — single `wishlist_items` table, `UNIQUE (userId, productId)`,
FK CASCADE on both `userId` and `productId`, index on `userId`. `apps/api/src/wishlist/`
mirrors `cart/`: `GET /wishlist`, `POST /wishlist/items`, `DELETE /wishlist/items/:productId`,
all behind the global `JwtAuthGuard`. Non-owned row → plain 404, never 403. `POST` is
idempotent (200, unchanged list; a concurrent-insert 23505 is swallowed) and deliberately does
**not** reject out-of-stock products. Price/currency/stock/image resolved live at read time.

**WL-2 — web service + heart toggle** (`MiiSqOS5`): `wishlist.service.ts`, signal-backed,
modelled on `CartService`; `has(productId)` backed by a derived `Set` for O(1) membership.
`ProductCard` stays presentational (`wishlisted` input / `wishlistToggle` output, no injected
service), wired into shop, discover, public vendor profile, and the PDP related-products grid.
Anonymous click → `/login?returnUrl=<current url>`. Hydration-gated with the `hydrated` signal
+ `afterNextRender` pattern from `nav-bar.ts`. No optimistic mutation.

**WL-3 — PDP toggle** (`RrgIdrIo`): toggle placed next to the sticky bar's add-to-cart CTA —
the PDP has no separate hero CTA (`pdp__hero` is the image gallery). Reuses WL-2's hydration
gate and `isWishlisted()`. The anonymous gate was extracted into one shared
`redirectAnonymous()` helper now used by add-to-cart and both wishlist toggles. Add-success
snackbar offers a "View wishlist" action to `/wishlist`.

**WL-4 — `/wishlist` page** (`zG9zNAh8`): `apps/web/src/app/features/wishlist/`,
`canActivate: [authGuard]`, `RenderMode.Client` (its guard depends on `localStorage`, so the
`**` → `RenderMode.Server` default would always bounce to `/login`). Loading/loaded/empty/error
states; out-of-stock rows keep their place with a badge and a disabled add-to-cart; removal
re-renders from the server response, never a local splice. `ProductCard` was **not** reused —
it takes `ProductDto`, a wishlist row is `WishlistItemDto`, and it has no remove/out-of-stock
affordance; the row is modelled on the cart line item instead.
`docs/design/wishlist/export.html` + `README.md` are hand-authored, not synced from Claude
Design (DesignSync needs an interactive login, unavailable this session) — same precedent set
by the vendor-profile screen.

**WL-5 — nav entry + badge** (`L54YO6fI`): radial-nav wishlist item gets
`routerLink: '/wishlist'` and stops emitting `itemSelected`; `shop.ts`'s "Wishlist is coming
soon" branch and its stale spec assertions were deleted. Badge markup went into the
`@if (item.routerLink)` branch, not the `@else` button branch where the cart badge lives —
giving the item a `routerLink` moves it across that boundary. Desktop nav-bar wishlist button
is icon-only, no count badge (owner-confirmed divergence from `nav-bar__cart-badge`), asserted
by a spec test so it isn't "fixed" later. `WishlistService.reset()` was added to
`AuthService.logout()` — wishlist rows are account-bound in Postgres and survive sign-out, so
this only clears in-memory state.

### Decisions taken during the batch, not already recorded above

1. **`<app-radial-nav>` is not `/shop`-only** — the spec's Out-of-scope claim was wrong; see
   the correction inline above. It renders on four templates (shop, discover, product-detail,
   vendor-profile), all already binding `[cartCount]`. `[wishlistCount]` was bound on all four
   — binding one page only would have shown the badge there and silently nowhere else.
   Radial-nav's page reach itself is unchanged.
2. **WL-5's Trello card contradicted the spec** on the desktop badge: it asked for a
   `nav-bar__wishlist-badge`, but the spec's resolved open question 4 says icon-only. The spec
   won — re-confirmed with the owner during this batch.
3. **WL-5's card assumed `CartService.reset()` was already wired into logout.** It isn't — only
   `checkout.ts` calls it, after order placement. Only the wishlist reset was added here; the
   pre-existing cart-badge-after-logout gap was deliberately left alone as a follow-up, not
   folded into this batch.
4. **The PDP has no separate hero CTA** — see WL-3 above; this shaped where the toggle landed.
5. **The `WishlistItems` migration is applied and verified** (updated 2026-08-10, later the
   same day — it was initially shipped unrun because the Docker daemon was unavailable).
   Run against Postgres 16 via Rancher Desktop: it was the only pending migration, and the
   resulting table matches the entity column-for-column (PK, `IDX_wishlist_items_userId`,
   `UQ_wishlist_items_userId_productId`, both FKs `ON DELETE CASCADE`). `down()` was exercised
   too — revert dropped the index and table with no residue, and a re-run reapplied cleanly.
   The idempotency backstop was confirmed directly: a duplicate `(userId, productId)` insert
   raises the SQLSTATE 23505 that `WishlistService.addItem` swallows on a concurrent add.

### Testing

`npm run lint:api` clean · `npm run test:api` 606 passed · `npm run test -w @hb/web` 777
passed · `npm run build` clean. Code review returned no FAIL items. A follow-up coverage audit
closed two gaps: hydration-gate tests (one per host page, asserting hearts read empty
pre-hydration) and the unique-violation race branch in `addItem`.
