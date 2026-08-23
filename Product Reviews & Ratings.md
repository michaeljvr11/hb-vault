# Product Reviews & Ratings

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Order State Machine]] · [[Auth & Roles]] · [[Public Storefront & SSR]] · [[Product Wishlist]] · [[Product Search Engine]] · [[Vendor Profile Customization]] · [[Money & Currency Rules]] · [[Customer Profile]]

## Problem

A shopper on the product-detail page has no social proof. There is no way to leave a
review, no way to read one, and no rating anywhere in the model. The vault has been
routing around this gap for months rather than filling it:

- [[Product Search Engine]] reserves `vendor_rating` as a **ranking slot that is
  deliberately inert**: "no rating model exists; a ratings/reviews feature is its own
  epic. Ranking rule reserved but inert until it lands." Its v1 ranking was cut to
  `inStock → recency → price` for exactly this reason.
- [[Vendor Profile Customization]] deferred **VPC-6** (`rule`-based auto sections) in part
  because "best sellers"/"top rated" need sales/rating signals that do not exist.
- [[Listing Types & Vendor Rules]] and `History/Josh/session-2.md` both record the
  "honest-to-data" decision that `VendorDto` carries **no rating**, so the Claude Design
  storefront's rating stars were omitted rather than faked.

Codebase state confirmed net-new: no `Review` entity, no `product_reviews` table, no
contract in `libs/shared/src/contracts/`, no reviews module in `apps/api/src`. On the web
side `product-detail.html` renders Description as a plain `pdp__section` — there is no tab
component anywhere in `apps/web` (`MatTabsModule` is unused; [[Customer Profile]]'s
`ProfileShell` "tabs" are actually child routes in a sidebar).

**Goal:** a customer who has actually received a product can rate it (1–5) and write a
short review; anyone — signed in or not — can read all reviews for that product in a
Reviews tab below the product description on the PDP, with the average rating, the review
count, and a **Verified Purchase** badge per review.

## Scope

**Backend** (`apps/api`)

1. `product_reviews` table + TypeORM migration.
2. `apps/api/src/reviews/` module (entity, controller(s), service, module, DTOs) mirroring
   the `wishlist/` file layout.
3. Endpoints:
   - `GET /products/:productId/reviews` — **`@Public()`** (the PDP is `RenderMode.Server`;
     this must render anonymously on the server). Paginated + summary.
   - `GET /products/:productId/reviews/eligibility` — authenticated. Tells the UI whether
     to show the submission form and why not, without making it POST and eat a 403.
   - `POST /products/:productId/reviews` — authenticated, verified-purchase gated.
   - `PATCH /reviews/:id` / `DELETE /reviews/:id` — author-only (optional v1 scope, see
     slices PR-5/PR-6).

**Frontend** (`apps/web`)

4. `core/api/reviews.service.ts` — signal-backed client, modelled on `WishlistService`.
5. A tabbed section on the PDP **below the Description section**, whose Reviews panel
   holds the summary header, the review list, and pagination.
6. The submission form inside that panel, rendered only for eligible purchasers.

## The verified-purchase question — RESOLVED

**Answer: yes — v1 hard-gates review submission on a verified purchase, and the qualifying
state is `OrderStatus.DELIVERED`.**

**Vault position:** no note in this vault states a rule about who may review. There is
therefore **no conflict to resolve** — every note that touches ratings says the same thing
(the feature is an unbuilt epic). This spec is the note that creates the rule.

**Rationale:**

1. **Industry norm (2026).** Gating submission to verified purchasers — or at minimum
   badging reviews as "Verified Purchase" — is the primary lever against fake reviews and
   the main driver of shopper trust in a review section
   (<https://wiserreview.com/blog/ecommerce-product-reviews/>,
   <https://contentsquare.com/guides/ecommerce-ux/product-review-section/>).
2. **The join already exists and is cheap.** `orders` carries `userId` + `status`;
   `order_items` carries `orderId` + `productId` (`apps/api/src/orders/entities/`).
   "Did user X buy product Y" is one join — no new tracking mechanism, no new column on
   orders, nothing to backfill.
3. **`delivered` is the only state that means the customer actually has the item.**
   Per [[Order State Machine]], `delivered` = "confirmed delivery to customer" and is
   **admin-only**. Everything earlier (`pending`, `confirmed`, `processing`,
   `handed_to_hb`, `shipped`) means the goods are still in the ZA→NA pipe. Reviewing a
   product you have not received is exactly what the gate exists to prevent, and on a
   cross-border route the gap between `shipped` and `delivered` is not a rounding error.
4. **`cancelled` never qualifies** regardless of what it was cancelled from.
5. **It composes with the settled auth rules.** [[Auth & Roles]] already requires
   `user.isVerified` to place an order (`OrdersService.create`, `orders.service.ts:154`),
   so a verified-purchase gate transitively inherits email verification for free. No
   second `isVerified` check is needed on the review endpoint.

**Operational dependency that must be flagged to a human:** `shipped → delivered` is a
**manual admin transition**, and today no vendor/admin portal UI calls
`PATCH /orders/:id/status` for it ([[Order State Machine]], slice VP-4 notes: portals
don't call it yet for admin-side transitions). If admins do not actually drive orders to
`delivered`, **zero customers will ever be eligible to review** and the feature will look
broken rather than empty. This is an ops prerequisite, not a code change in this epic —
but it must be true before launch. See open question 4.

## Business rules it must honour

From [[Auth & Roles]], [[Public Storefront & SSR]], [[Order State Machine]] and
[[Money & Currency Rules]] — enforce, do not reinvent:

- **Browse is public, transact is authenticated.** Reading reviews is browsing: public,
  anonymous, server-rendered. Writing one is transacting: authenticated. An anonymous
  "Write a review" click routes to `/login?returnUrl=<current url>` via the established
  `sanitizeReturnUrl` path — never a silent no-op, never a surfaced 401.
- **Auth is settled.** Global `JwtAuthGuard` + `@Public()`/`@Roles`. Extend, never fork.
  Note the `public-routes.guardrail.spec.ts` pin: adding a `@Public()` route requires
  updating that guardrail spec in the same PR or CI fails.
- **Ownership checks live in the service layer, and a non-owned row is a plain 404, never
  a 403** — the `OrdersService.findOneForUser` / `CartService.findOwnedItem` pattern. No
  existence leak. Applies to `PATCH`/`DELETE /reviews/:id`.
- **No PII leak via a public DTO.** The review list is anonymous-readable, so it must
  never expose the reviewer's email or full identity. See the `authorName` shape decision
  below. Same rule that forced `VendorResponseDto` over `AdminVendorResponseDto`.
- **Order state is read, never written.** This feature reads `orders.status`; it does not
  add a state, a transition, or a status write. `ORDER_STATUS_TRANSITIONS` is untouched.
- **Order/inventory-adjacent logic ⇒ unit tests in the same PR** (repo non-negotiable).
  The eligibility join reads order state, so it carries the full test burden — see PR-2.
- **Schema change ⇒ TypeORM migration**, `synchronize` stays off, `down()` drops cleanly.
  New table so there is no backfill/DEFAULT concern.
- **Every endpoint input ⇒ a class-validator DTO implementing the shared interface.**
  Query params (`page`/`limit`/`sort`) included.
- **SSR safety is non-negotiable.** The PDP is `RenderMode.Server`. The review *list* must
  server-render (it is SEO-relevant content). Everything auth-dependent — the eligibility
  call, the form, the edit/delete controls — must sit behind the `hydrated` signal +
  `afterNextRender` gate from `nav-bar.ts`, or the first client render diverges from the
  server DOM.
- **No money here.** Ratings are not money; [[Money & Currency Rules]] is untouched. No
  price is snapshotted onto a review, and there is no currency on this entity.

## Shape decisions taken (flag if wrong — do not silently change)

1. **Rating is an integer 1–5**, stored `smallint` with a DB `CHECK (rating BETWEEN 1 AND 5)`
   *and* class-validator `@IsInt() @Min(1) @Max(5)`. Not 1–10, not half-stars, not a
   decimal. Five stars is what shoppers parse without a legend and what every competing
   storefront trains them on; a 10-point scale collapses into "8+ = good" in practice.
   See open question 1.
2. **Body text is required**, min 10 / max 2000 characters, plain text (no markdown, no
   HTML — rendered as escaped text, `\n` split into paragraphs like `pdp__description`).
   The user's ask was "rating + text", so rating-only reviews are not accepted in v1.
3. **No review title in v1.** The ask was rating + text. A `title` field is a follow-up,
   not an assumption.
4. **`UNIQUE (productId, userId)` — one review per user per product**, mirroring
   `wishlist_items`' unique constraint. A second `POST` returns **409**, not a silent
   upsert — deliberately unlike the wishlist's idempotent `POST`, because a review carries
   content that would be destroyed by a silent overwrite. Buying the same product twice
   still yields one review. See open question 2.
5. **Verified-purchase status is a boolean snapshot column (`isVerifiedPurchase`), not a
   live join and not an FK to `order_items`.** Written `true` at submission time by the
   eligibility check. Rationale: this repo already snapshots facts that must not be
   restated later (`order_items.unitPrice`, `productName`, `commissionRatePercent`). An FK
   to `order_items` was considered and rejected — orders get cleaned up, and a badge that
   silently flips to false because a row was deleted is worse than a stale-but-honest one.
   In v1 the column is always `true` (the gate guarantees it); it exists so that if the
   gate is ever relaxed to open reviews, the badge stays meaningful and historically correct.
6. **`authorName` is a derived display name, never stored and never the email.**
   `firstName` + last-name initial (`"Michael J."`), falling back to `firstName` alone,
   falling back to `"H&B Customer"` when the user has no name. Computed at read time in
   the service, exactly like the wishlist's live-resolved product fields.
7. **A vendor may not review their own listing.** If the caller's `vendorId` matches
   `product.vendorId`, submission is refused (403) even if they somehow hold a delivered
   order for it. Cheap to enforce, closes an obvious self-dealing hole in the marketplace
   model ([[Listing Types & Vendor Rules]]).
8. **The summary is computed with a SQL aggregate at read time — no denormalised
   `ratingSum`/`ratingCount` columns on `products` in v1.** Review volume at launch is
   low; a counter cache is a cache-invalidation problem to buy later, once the average
   needs to appear on product cards and in search ranking (both out of scope, below).
9. **`averageRating` is rounded to one decimal** (`4.3`), computed over integers. Assert
   the rounding in a unit test — it is aggregate arithmetic on user-visible numbers, and
   the repo's habit is to test arithmetic rather than eyeball it.
10. **Reviews tab = a new tabbed section on the PDP, not a route and not
    `MatTabsModule`.** `apps/web` has no tab component today; `ProfileShell`'s "tabs" are
    child routes, which is wrong here (a deep-linked `/products/:id/reviews` route would
    fragment the PDP's SSR payload). Hand-rolled tab strip with `role="tablist"` /
    `aria-selected`, styled from `docs/design/DESIGN.md` tokens, matching the existing
    `pdp__section` rhythm. v1 ships **one** tab ("Reviews (N)") in a tab-group structure so
    Q&A / Shipping tabs can join it later without a rewrite. See open question 3.
11. **Reviews sort newest-first by default**, page size 10, `?sort=newest|highest|lowest`.
12. **Deleting a product or a user cascades the reviews away** (`ON DELETE CASCADE` on
    both FKs), mirroring `wishlist_items`. No tombstone rows — `Product` still has no
    soft-delete/publish flag ([[Product Wishlist]] out-of-scope note).

## `@hb/shared` contract impact

New `libs/shared/src/contracts/review.ts`, exported from `contracts/index.ts` (interfaces
only — DTO **classes** live in `apps/api` and `implement` these):

```
ReviewDto {
  id; productId; rating: number; body: string;
  authorName: string;            // derived, never the email — shape decision 6
  isVerifiedPurchase: boolean;   // snapshot — shape decision 5
  createdAt: string; updatedAt: string;
}

ProductReviewSummaryDto {
  averageRating: number | null;  // null when reviewCount === 0 — never 0, which reads as "rated 0 stars"
  reviewCount: number;
}

ProductReviewListDto extends PagedResponse<ReviewDto> {
  summary: ProductReviewSummaryDto;
}

ProductReviewQuery { page?; limit?; sort?: ReviewSort }
CreateReviewRequest { rating: number; body: string }
UpdateReviewRequest = Partial<CreateReviewRequest>

ReviewEligibilityDto {
  canReview: boolean;
  reason: ReviewIneligibilityReason | null;   // null when canReview
  existingReviewId?: string;                  // set when reason === 'already_reviewed'
}
```

Conventions to follow:

- `PagedResponse<T>` from `contracts/common.ts` is the **established list envelope**
  (`ProductsService.findAll` returns `PagedResponse<ProductDto>`). Extend it rather than
  inventing a fourth pagination shape — note `ProductSearchResponse` in `contracts/search.ts`
  uses `pageSize` rather than `limit`; **follow `PagedResponse` (`limit`), not search.**
- New enum file `libs/shared/src/enums/review-sort.ts`, exported from `enums/index.ts`,
  using the settled **const-object + type** pattern (`order-status.ts`, `country.ts`) —
  **not** a TypeScript `enum`:
  `ReviewSort = { NEWEST: 'newest', HIGHEST: 'highest', LOWEST: 'lowest' }`.
- `ReviewIneligibilityReason` in the same file:
  `{ NOT_PURCHASED: 'not_purchased', ALREADY_REVIEWED: 'already_reviewed', OWN_LISTING: 'own_listing' }`.
  (`not_authenticated` is not a value — an anonymous caller gets a 401 from the global
  guard and the UI shows the sign-in CTA without asking.)
- `ProductSort` is a bare string union in `contracts/product.ts` while
  `ProductSearchSort` is an enum file. `ReviewSort` goes in `enums/` because the API
  validates it with `@IsEnum` and it is a fixed, shared vocabulary.

**Unchanged:** `ProductDto` gains **no** `ratingAverage`/`ratingCount` in v1 (see out of
scope). `AnalyticsEventType` gains nothing. `OrderStatus` gains nothing. `CartDto`,
`WishlistDto`, `VendorDto` untouched.

## Data model

`product_reviews`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `productId` | uuid FK → `products` | `ON DELETE CASCADE`, indexed |
| `userId` | uuid FK → `users` | `ON DELETE CASCADE` |
| `rating` | `smallint` | `CHECK (rating BETWEEN 1 AND 5)` |
| `body` | `text` | length enforced by the DTO |
| `isVerifiedPurchase` | `boolean NOT NULL DEFAULT true` | snapshot, shape decision 5 |
| `createdAt` / `updatedAt` | `timestamptz` | |

Constraints/indexes: `UNIQUE (productId, userId)`, index on `(productId, createdAt DESC)`
for the default list query. New table, so no backfill or column DEFAULT hazard.

**The eligibility join** (no new schema, read-only):

```
EXISTS (
  SELECT 1 FROM order_items oi
  JOIN orders o ON o.id = oi."orderId"
  WHERE oi."productId" = :productId
    AND o."userId"     = :userId
    AND o.status       = 'delivered'
)
```

## Out of scope (v1) — explicitly not built, each its own future card

- **Moderation, reporting, profanity filtering, admin takedown.** Reviews publish
  immediately. There is no `status`/`approved` column and no admin queue. The
  verified-purchase gate is the only abuse control in v1. If a review needs removing,
  it happens in the database.
- **Vendor responses / replies to reviews.** No thread model.
- **Photo or video uploads on reviews.** Would need the [[Product Image Optimization Pipeline]]
  work first (PIO-1..PIO-5 are still open cards).
- **Helpful / unhelpful voting, sorting by helpfulness.**
- **Average rating on `ProductDto`, product cards, or the discovery grid.** Adding it means
  aggregating on every product list read — buy the counter cache (shape decision 8) first.
- **Search ranking.** [[Product Search Engine]] reserves a `vendor_rating` slot. Note the
  mismatch to surface, not paper over: this epic produces a **product**-level rating,
  while that slot expects a **vendor**-level one. A vendor rating would be a derived
  aggregate across that vendor's products' reviews — a separate card, and a separate
  decision about whether it is even the right signal. Nothing here wires Meilisearch.
- **VPC-6 "top rated" auto sections** ([[Vendor Profile Customization]]) — unblocked by
  this epic in principle, not delivered by it.
- **Rating filter facet** ("4 stars & up") and the star-distribution histogram bars.
  Summary is average + count only.
- **Analytics events.** `AnalyticsEventType` is documented as "intentionally final …
  without a fresh review", so no `REVIEW_SUBMITTED` event — same precedent [[Product Wishlist]] set.
- **"Rate your purchase" emails.** [[Transactional Email & Order Notifications]] would own
  a post-delivery prompt; nothing here sends mail.
- **A "My reviews" tab in [[Customer Profile]]** and any review surface outside the PDP.
- **Review title field, half-star ratings, anonymous/pseudonymous reviewing.**
- **Backfilling reviews or importing them from anywhere.**

## Open questions (a human must confirm — the stated default stands if nobody objects)

1. **Rating scale: 1–5 integer stars?** Default: **yes, 1–5 integers** (shape decision 1).
   Changing this after PR-1 means a contract change, a `CHECK` constraint migration and a
   data migration of existing ratings — cheap now, expensive later. **Confirm before PR-1.**
2. **One review per user per product, enforced by `UNIQUE (productId, userId)`?**
   Default: **yes**, second `POST` → 409. The alternative (one review per *delivered order
   line*, so a repeat buyer can review twice) is defensible but complicates both the
   constraint and the UI. **Confirm before PR-1** — it is a schema constraint.
3. **PDP tab structure.** v1 adds a tab group below the Description section containing a
   single "Reviews (N)" tab. Should the existing **Description** and **Cross-Border
   Shipping** sections move *into* that tab group (Description | Shipping | Reviews), which
   is the more conventional PDP layout? That is a broader PDP restructure and a design
   decision, so v1 does **not** do it. Confirm the one-tab-for-now reading of "in a tab",
   or greenlight the restructure as its own design + card.
4. **Ops prerequisite: will orders actually reach `delivered`?** The gate is only as good
   as the admin transition that feeds it, and no portal UI drives `shipped → delivered`
   today ([[Order State Machine]]). Either (a) accept that reviews stay empty until admins
   drive that transition via the API, or (b) raise an admin-order-actions card as a
   blocker for this epic. **This is the highest-risk item in the spec.**
5. **Are edit/delete of one's own review in v1?** Default: **yes** — slices PR-5/PR-6,
   deliberately placed last so they can be cut without disturbing PR-1..PR-4. Note the
   trade-off if cut: with the `UNIQUE` constraint from decision 4, a user who submits a
   typo is stuck with it forever and has no recourse. Recommend keeping them.
6. **Should an edited review be marked "Edited"?** Only relevant if 5 is yes. Default:
   **no visible edited marker in v1** (`updatedAt` is on the contract, so the UI can add
   one later without a contract change).


**Update 2026-08-16 — question 4 resolved, scope grew.** Rather than a narrow "mark
delivered" admin action, the human confirmed admin should get a fully unrestricted
order-status override (any status → any status, reason required, audited) — see
[[Order State Machine]] § "Admin override (confirmed 2026-08-16)". This supersedes the
originally-scoped blocking card; the override mechanism covers the `shipped → delivered`
case as one instance of its general capability. New blocking cards track the override
API + admin UI in place of the original card.

## Vertical slices → Trello cards
| # | Title | Depends on | Card |
|---|---|---|---|
| PR-1 | `@hb/shared` review contracts + `product_reviews` migration + public read API | — | [#117](https://trello.com/c/4loUsIJ7) |
| PR-2 | Verified-purchase eligibility + `POST` review API (+ order-join unit tests) | PR-1 | [#118](https://trello.com/c/3pAC43jY) |
| PR-3 | Web: PDP Reviews tab — summary, list, pagination, Verified Purchase badge | PR-1 | [#119](https://trello.com/c/XJjlnz9y) |
| PR-4 | Web: review submission form on the PDP, eligibility-gated | PR-2, PR-3 | [#120](https://trello.com/c/saPh1Fmu) |
| PR-5 | API: edit/delete own review (optional scope — see open question 5) | PR-2 | [#121](https://trello.com/c/23PYFGW7) |
| PR-6 | Web: edit/delete own review from the PDP (optional scope) | PR-4, PR-5 | [#122](https://trello.com/c/7XSMAVih) |

Order: PR-1 → PR-2 → PR-3 → PR-4 → PR-5 → PR-6. PR-3 can run in parallel with PR-2
(it only needs PR-1's read endpoint). PR-5/PR-6 are the droppable tail.

Branch names follow `feat/<card-shortLink>-<slug>`, e.g. `feat/4loUsIJ7-review-contracts-read-api`.

## Implementation Notes (2026-08-23)

Shipped as four consecutive slices on one branch, `feat/4loUsIJ7-review-contracts-read-api`, bundled as one PR (all four cards share the review contract, each subsequent slice depends on an earlier one). PR not yet opened at time of writing.

**PR-1 — API contracts + migration + public read** (`0b9746e`): `libs/shared/src/contracts/review.ts` with `ReviewDto`, `ProductReviewSummaryDto`, `ProductReviewListDto`, `ProductReviewQuery`, `CreateReviewRequest`, `UpdateReviewRequest`, `ReviewEligibilityDto`. `libs/shared/src/enums/review-sort.ts` with `ReviewSort` const-object (`NEWEST`, `HIGHEST`, `LOWEST`) and `ReviewIneligibilityReason` enum. Migration `<timestamp>-ProductReviews.ts` creates `product_reviews` table: `id` (uuid PK), `productId`/`userId` (FKs with `ON DELETE CASCADE`), `rating` (smallint `CHECK (rating BETWEEN 1 AND 5)`), `body` (text), `isVerifiedPurchase` (boolean, always `true` in v1), `createdAt`/`updatedAt` (timestamptz), `UNIQUE (productId, userId)`, index on `(productId, createdAt DESC)`. `apps/api/src/reviews/reviews.controller.ts` exposes `GET /products/:productId/reviews` with `@Public()` — anonymous-readable, paginated (10 per page, default sort newest-first), returns `ProductReviewListDto` with SQL-aggregated `ProductReviewSummaryDto` (averageRating rounded to 1 decimal, null when reviewCount === 0; reviewCount). `ReviewsService` computes `authorName` (derived: firstName + last-initial, fallback to firstName alone, fallback to "H&B Customer") at read time, never stored. No PII leak to the public DTO.

**PR-2 — Eligibility gate + POST** (`42ef625`): `GET /products/:productId/reviews/eligibility` (authenticated) returns `ReviewEligibilityDto` with `canReview: boolean` and `reason: ReviewIneligibilityReason | null` (one of: `not_purchased`, `already_reviewed`, `own_listing`; null when eligible). Eligibility is an EXISTS join over `order_items`/`orders` requiring `OrderStatus.DELIVERED`. `POST /products/:productId/reviews` (authenticated, requires verified eligibility) creates a review, writes `isVerifiedPurchase: true` at submission time as a permanent snapshot. Duplicate review (same user + product) returns 409 per the `UNIQUE` constraint. Vendor-owns-listing (caller's `vendorId` matches `product.vendorId`) returns 403 even if the user holds a delivered order. Unit tests cover the eligibility join (DELIVERED required, other statuses ineligible), duplicate detection (409), and vendor self-review block.

**PR-3 — Web reviews tab + pagination** (`5c9651b`): `apps/web/src/app/core/api/reviews.service.ts` (signal-backed, modelled on `WishlistService`) provides `reviewList`, `loading`, `error` signals and methods to fetch/paginate. Hand-rolled tab strip on the PDP (not `MatTabsModule`, following the vault decision to reserve tabs for future Q&A/Shipping sections) below the Description section with `role="tablist"` / `aria-selected` semantics, styled from design tokens. Server-rendered review list (SEO-relevant, `RenderMode.Server` on the PDP) with summary header (average rating + review count + Verified Purchase badge per review). Pagination controls (page/limit), newest-first default. Null averageRating never renders as zero. The tab group structure ships with one tab ("Reviews (N)") so Q&A/Shipping tabs can join it later without a rewrite.

**PR-4 — Web review submission form** (`6d30984`): Reactive form on the PDP, hydration-gated (`hydrated` signal + `afterNextRender`, mirroring `nav-bar.ts`) because eligibility state is auth-dependent and must not diverge from the server render. Rating 1-5 radio buttons + body textarea (required, 10-2000 chars). Rendered only when `canReview: true`. Distinct messaging per ineligibility reason (not purchased, already reviewed, own listing). Anonymous click routes to `/login?returnUrl=<current url>` via `sanitizeReturnUrl`. On successful submission, post-success eligibility refetch (now returns `already_reviewed`) triggers form unmount — fix (commit `6e169bc`) hoisted the success confirmation ("Thanks — your review has been posted") to a sibling of the eligibility branches so the user sees it before the form disappears. DTO whitespace-body validation gap fixed: `@Length(10, 2000)` let a whitespace-only body through untrimmed; added the repo's existing `trimmed()` Transform pattern (from `create-contact-inquiry.dto.ts`) to both `CreateReviewDto` and the Angular form control.

**PR-5 — API: author-only edit/delete** (`ada3f7d` + `ab01535`): New flat `@Controller('reviews')` routes `PATCH /reviews/:id` and `DELETE /reviews/:id` (204 no body), deliberately separate from the existing product-scoped `ReviewsController` to match the spec's `/reviews/:id` shape. Resolves a spec/contract mismatch: spec said flat endpoint, `libs/shared/src/contracts/review.ts` JSDoc incorrectly stated nested route — spec followed as truth, JSDoc corrected. `ReviewsService.findOwnedReview` (private) scopes all lookups to `{id, userId}` and throws a plain 404 on non-owned or unknown review (no 403 existence leak, mirroring `OrdersService.findOneForUser`). `UpdateReviewDto implements UpdateReviewRequest` — `rating` and `body` both optional with same bounds as create (1–5 int, 10–2000 chars trimmed), but empty `{}` PATCH body explicitly rejected (400 via `@ValidateIf` guard). Immutability enforced structurally: service spreads only `rating`/`body` onto the entity, never the DTO itself, plus global `whitelist: true` ensures `productId`/`userId`/`isVerifiedPurchase` cannot be mutated from the request. Code review catch: malformed `:id` UUID produced unhandled 22P02 Postgres error (500) — fixed with `ParseUUIDPipe` on both route params (commit `ab01535`). Tests: `reviews.service.spec.ts` extended to 72 in-file tests covering owner edit/delete, non-owner 404, unknown-id 404, validation bounds, empty-body rejection, immutability, and summary recompute after edit/delete (including edge case: last review deleted → `averageRating` reverts to `null`).

**PR-6 — Web: edit/delete UI on PDP** (`0b4dbac`): Edit/Delete controls added to the `.pdp__review-existing` card (the one rendering the authenticated caller's own review under the `already_reviewed` eligibility branch) — never added to the general reviews list, so no other reviewer's row exposes controls. Edit reuses the exact `reviewForm`/`setRating()` signals from PR-4's create flow, pre-filled with current rating/body. An attempt to share the star-rating/textarea markup via `<ng-template>` + `*ngTemplateOutlet` hit Angular `NG01050` DI resolution bug (the outlet keeps the declaration-site injector, not insertion-site, breaking `formControlName` binding in reactive forms) — reverted to duplicating ~60-line markup block (worth remembering for future template-sharing attempts in reactive-forms contexts). Delete uses an inline "Are you sure?" confirm, not `window.confirm()` (SSR/testability). Both mutations always re-fetch `loadReviews()` + `fetchEligibility()` after success — never local splice — same discipline as PR-4's `submitReview()`. Code review catches: (1) error message (e.g. 404 on stale edit/delete after review removed by another user) was nested inside eligibility branches, so the refetch it triggered would unmount it before the user read it — **same class as the PR-4 success-confirmation bug** — hoisted error block to sibling of `reviewSubmitSuccess()`, outside eligibility branches; (2) two Vitest 404 tests asserted error message rendered but mocked refetch still returned stale "already_reviewed" state, masking whether the hoist worked — tests passed even before fix — fixed by making mocked refetch agree with the error (eligibility flipped back to `canReview: true`). No "Edited" marker rendered (spec open question 6 default upheld). Tests: `product-detail.spec.ts` extended with edit/delete describe block; 930/930 passing.

### Confirmed decisions from the spec

1. **Rating scale 1–5 integer** — shipped exactly as stated, no deviation. `smallint` with `CHECK`, class-validator `@IsInt() @Min(1) @Max(5)`.
2. **`UNIQUE (productId, userId)` enforced at DB + API** — duplicate `POST` returns 409, not idempotent. No silent upsert (unlike wishlist), because review content would be destroyed by overwrite.

### Lessons from the batch

**Eligibility-state-refetch-unmounting pattern (recurred in both PR-4 and PR-6).** Success/error messaging nested inside eligibility branches unmounts when a refetch re-evaluates the branch condition. In PR-4, the post-submit eligibility refetch (confirming `already_reviewed`) unmounted the success confirmation. In PR-6, the edit/delete error message (on 404 from a stale request) was nested inside `canReview` branches, so the refetch triggered by the error unmounted the message before the user could read it. Both fixes hoisted the message to a sibling of the eligibility branches. This pattern matters for any eligibility-gated feature that shows immediate feedback (checkout forms, upload flows, etc.): success/error must sit outside the eligibility-dependent branch, or be paired with a stale-state indicator that signals to the user their action completed and the state refresh succeeded.

### Scope boundaries

- **PR-5/PR-6 (edit/delete own review) were explicitly optional but built per human confirmation** (open question 5: "recommend keeping them"). Without edit/delete, a user with a typo would be permanently stuck due to the `UNIQUE(productId, userId)` constraint — no recourse. The final delivery includes these slices.
- **No admin moderation, profanity filtering, takedown, or review status column** — v1 publishes immediately.
- **No denormalised rating on `ProductDto`, product cards, or search** — aggregated at read time only; counter cache deferred.
- **`product.rating` slot in ranking is untouched** — spec notes the mismatch (product rating vs. vendor rating) for a future decision.
- **No "Edited" marker** on edited reviews (spec open question 6 default upheld; `updatedAt` on the contract for future use).

### Testing

`npm run lint:api` clean · `npm run test:api` **860/860** · `npm run test -w @hb/web` **930/930** · `npm run build` clean (pre-existing SCSS budget warnings in `product-detail.scss` now 7.24kB over budget due to edit/delete markup duplication — flagged as minor follow-up, not fixed, out of scope). Code review (pre-PR) caught three issues: (1) PR-5 malformed UUID → unhandled 500, fixed with `ParseUUIDPipe`; (2) PR-6 error message nesting → unmount-on-refetch, hoisted outside eligibility branches; (3) PR-6 test mocks stale after error, now agree with error state.
