# Storefront UI Cleanup Batch
Status: **shipped 2026-08-12**, PR open, awaiting human merge.
Related: [[Public Storefront & SSR]] · [[Listing Types & Vendor Rules]] · [[Category Taxonomy & Discovery]] · [[HB Domain Model]]

## Problem

Five customer-facing defects in `apps/web` were reported together. They are all cosmetic
or content-level (no API, no schema, no money/inventory/order logic), but three of them
share a single root cause, so they are sliced by cause rather than by symptom.

1. **Snackbar is unreadable and misplaced.** Every `MatSnackBar.open()` call site passes
   `verticalPosition: 'top'` + `horizontalPosition: 'end'`, so the toast renders in the
   top-right and overlaps the sticky nav bar. Options (duration, position, `panelClass`)
   are copy-pasted across 8 components, so the styling drifts and there is no single
   place to fix it.
2. **"SME Verified" is a redundant claim.** Product cards, the PDP, the vendor profile
   and the vendor showcase all render a verification badge, and `/discover` has a
   "Verified SME Vendors" filter toggle. Per the platform rule, every publicly visible
   vendor is already approved (vetted), so the badge asserts something that is true of
   100% of what a customer can see — it carries no information.
3. **Horizontal scroll on desktop.** The page scrolls left/right at laptop widths.
4. **PDP "Add to Cart" bar oversized / cut off** at smaller laptop widths.
5. **Dead nav items.** "SME Directory" and "Logistics" in the top nav lead nowhere — they
   only fire the "coming soon" toast. They are not planned features.

## Root cause found (items 3 + 4)

There is **no global `box-sizing: border-box` reset**. `apps/web/src/styles.scss` sets
`html, body { margin: 0; ... }` and nothing else; only nine unrelated components opt in
to `border-box` locally. Everything else is browser-default `content-box`, so any rule
that combines `width: 100%` with horizontal padding is **wider than its container by the
padding**:

- `apps/web/src/app/layout/nav-bar/nav-bar.scss` — `.nav-bar__inner { width: 100%; max-width: 1280px; padding: 16px 40px; }`
  → 1280 + 80 = **1360px**, i.e. it overflows every viewport narrower than 1360px.
  That is exactly the 1280×800 / 1366×768 laptop band where the bug was reported.
- `apps/web/src/app/features/discover/discover.scss` — `.discover { width: 100%; max-width: 1280px; padding: 24px 40px 48px; }` — same 80px overflow.
- `apps/web/src/app/features/shop/shop.scss` — `.section { width: 100%; max-width: 1280px; padding: 48px 40px; }` — same.
- `apps/web/src/app/features/product-detail/product-detail.scss` — `.pdp__sticky-bar { position: fixed; left: 0; width: 100%; padding: 8px 40px; }`
  → the fixed bar is 80px wider than the viewport, which **is** defect 4: the bar has no
  `max-width`, the button is `flex: 1`, so "Add to Cart" stretches to nearly the full
  viewport and its right edge is pushed off-screen.

So items 3 and 4 are the same bug seen twice. Fixing the box model fixes both; the PDP
bar additionally needs a `max-width` so the button stops being a full-bleed slab on
desktop (the design intends this bar as a mobile pattern).

## Business rule: vendor verification

**Confirmed in the vault**, [[Listing Types & Vendor Rules]]:

- Vendor lifecycle is `pending → approved`, `pending → rejected`, `approved ↔ suspended`.
- "Only `approved` vendors can list products and receive orders (business rule — enforce
  in the service layer)."
- The public vendor directory endpoint returns approved vendors only; the approved-only
  filter was extended to the product catalogue (card #36 `c2o6xfZs`), so pending/rejected/
  suspended vendors never reach a customer surface at all.

**Therefore the customer-facing verification badge is provably redundant** — the customer
cannot be shown an unverified vendor by construction. `verificationDocumentUrl` / KYC
review exists on the entity but is explicitly a *future* card and is not v1 (same note,
"V1 onboarding depth"). Verification remains an **internal admin concept**; the admin
portal's own vendor-status UI is unaffected by this batch.

**Nuance worth flagging** (see Open questions): the badge and the `/discover` toggle are
not actually keyed on verification status. They are keyed on `product.vendor` being
present:

- `apps/web/src/app/shared/components/product-card/product-card.html` — `@if (product().vendor) { … SME VERIFIED }`
- `apps/web/src/app/features/discover/discover.ts` — `smeOnly = signal(true)`, `filteredProducts = products.filter(p => !!p.vendor)`

So they really distinguish **marketplace vendor listings from platform-fulfilled
listings**, mislabelled as verification. And because `smeOnly` defaults to **`true`**,
platform-fulfilled listings are currently hidden on `/discover` by default. Deleting the
toggle therefore *changes what customers see* — it is not a pure cosmetic removal.

## Scope

### 1 — Remove dead nav + redundant verification concept (customer-facing only)

Nav / dead placeholders:
- `apps/web/src/app/layout/nav-bar/nav-bar.html` — remove the "SME Directory" and
  "Logistics" buttons (lines 18–27). The `notifyComingSoon()` method stays: the currency
  switcher (line 44) still uses it.
- `apps/web/src/app/features/shop/shop.html` line 41–45 — the "SME Verification" hero CTA
  (a ghost button that only fires the toast) and its `onHeroSmeVerification()` handler in
  `shop.ts` line ~168.
- `apps/web/src/app/layout/footer/footer.html` — "SME Verification" (line 20) and
  "Logistics Partnership" (line 29) dead `href="#"` links.

Verification badge / filter / wording:
- `shared/components/product-card/product-card.html` lines 20–25 — SME VERIFIED badge
  (+ its `.product-card__badge*` SCSS, + `product-card.spec.ts` line ~68).
- `features/product-detail/product-detail.html` lines 50–55 — "SME Verified" hero badge
  (+ `.pdp__sme-badge` SCSS, + the related PDP spec assertion).
- `features/vendors/vendor-profile/vendor-profile.html` lines 35–36 — "Verified SME" badge.
- `shared/components/vendor-showcase/vendor-showcase.html` lines 14–20 + `isVerified()` in
  `vendor-showcase.ts` line 45 + `.vendor-card__verified` SCSS + `vendor-showcase.spec.ts` line 49.
- `features/discover/discover.html` lines 33–50 — the whole "Verified SME Vendors" toggle
  block, `.discover__sme-toggle` / `.discover__switch` SCSS, `smeOnly` + `filteredProducts`
  + `onSmeToggle` in `discover.ts`, and `discover.spec.ts` line ~414.
- Copy: `shop.html` line 8 placeholder "Search **verified** SME products…", `shop.html`
  line 32 "Connecting **verified** South African SMEs…".
- Token `--hb-sme-badge-bg` in `styles.scss` line 24 becomes dead — remove if unreferenced.

**"SME" stays.** Only the *verification claim* goes. "Featured SME Vendors", "SME Access",
"SME partners" describe small/medium enterprises and remain valid copy.

### 2 — Centralise + restyle the snackbar

Every call site currently repeats position/panelClass. Call sites found:
`layout/nav-bar/nav-bar.ts:68`, `features/shop/shop.ts:234,244,255,264`,
`features/discover/discover.ts:311,321,347`, `features/product-detail/product-detail.ts:237,247,269,297,306`,
`features/vendors/vendor-profile/vendor-profile.ts:234,244,270`, `features/cart/cart.ts:81`,
`features/wishlist/wishlist.ts:77`, `auth/login/login.ts:85,94`, `auth/register/register.ts:101`.

Target: one shared notification helper/service under `apps/web/src/app/core/` owning
default `MatSnackBarConfig` (bottom placement, duration, panel class per severity), with
success/info/error entry points. The three global panel classes already exist in
`styles.scss` lines 326–343 — the success class `#015300` on `#ffffff` is the green
confirmation style asked for; it just needs to become the default for confirmations and
to be applied at the bottom of the screen.

### 3 — Box model / layout overflow

Global `box-sizing: border-box` reset in `styles.scss`, plus a `max-width` on the PDP
sticky bar and a right-sized Add-to-Cart button.

## Data / contract impact

**None.** No `@hb/shared` change, no DTO, no endpoint, no migration, no money/inventory/
order-state logic. Existing Vitest specs that assert the removed badge/toggle must be
deleted or rewritten in the same PR — that is the test obligation for this batch.

## Out of scope

- Admin portal verification UI (`admin-vendors`, `admin-users` "Verified" badges) — that
  is the internal concept and stays.
- Vendor onboarding copy ("Apply to become a verified vendor") — describes the vetting
  process to a prospective *vendor*, not a claim to a customer. Leave it.
- `trust-banner` "zero cross-border fees for verified users" — refers to *user* email
  verification, unrelated to vendor vetting. Leave it.
- The remaining `href="#"` placeholder footer links and other `notifyComingSoon` targets
  (Currency switcher, Newsletter, Support, Terms of Trade, Privacy Policy, My Orders) —
  those represent real intended features, unlike SME Directory / Logistics.
- Building an SME Directory or Logistics page. The request is removal, not deferral.
- Dark mode. The app has no dark theme (no `prefers-color-scheme` anywhere in `apps/web`).
- Rebuilding the PDP sticky bar as a desktop-only or scroll-aware component. Sizing fix only.

## Open questions (confirm before /ship-card)

1. **`/discover` default result set changes.** `smeOnly` defaults to `true`, so platform-
   fulfilled listings are hidden today. Removing the toggle means every customer sees
   platform + vendor listings by default for the first time.
   *Recommendation: yes, show everything — the current default silently hides half the
   catalogue.* Needs a human yes/no because it changes what customers see, not just how.
   **RESOLVED** — confirmed with Michael 2026-08-11.
2. **Do we keep any vendor-vs-platform signal on a product card?** The badge was the only
   visual marker. The PDP keeps its vendor card ("View Store"), so PDP is fine; grid cards
   would lose the distinction entirely.
   *Recommendation: accept the loss for now; if we want it back, it should be an honest
   "Sold by <vendor>" label, not a verification badge — separate card.*
3. **If KYC / `verificationDocumentUrl` review ships later, does the badge come back?**
   *Recommendation: no. Verification stays internal; vetting is a precondition of being
   listed, not a differentiator between listings.*
4. **Global `box-sizing: border-box` is the correct fix but has a wide blast radius** —
   it changes the box model for the vendor portal, admin portal and auth screens too,
   some of which were laid out against `content-box` and may shift.
   *Recommendation: do the global reset (it is the right long-term fix) and require a
   visual pass over storefront + discover + PDP + cart + checkout + vendor portal + admin
   in the same PR. Alternative if that proves too noisy: scope `border-box` to the
   customer-facing containers only, and note the debt here.*
   **RESOLVED** — blast radius measured and verified benign; all 9 affected surfaces (admin/profile/vendor sidebars, auth cards, synonym rows) inspected and confirmed zero clipping.
5. **Footer dead links** — this batch removes only "SME Verification" and "Logistics
   Partnership". The other `href="#"` footer links stay dead. Confirm that is acceptable
   or raise a separate card.

## Vertical slices (→ Trello cards)

1. **UIC-1 — Remove dead nav items + the "SME Verified" concept from the storefront.**
   Card #85 `IP59Crue` — https://trello.com/c/IP59Crue
   Content/removal slice. Touches `nav-bar`, `shop`, `discover`, `product-card`,
   `product-detail`, `vendor-profile`, `vendor-showcase`, `footer`.
   *Gated on open question 1 (the `/discover` default-result-set change).*
2. **UIC-2 — Centralise snackbar config; bottom placement, legible Material styling.**
   Card #86 `7sclIgtI` — https://trello.com/c/7sclIgtI
   Touches every snackbar call site. Run **after** UIC-1 (UIC-1 deletes two call sites in
   `nav-bar.html` and one in `shop.ts`).
3. **UIC-3 — Fix the box-model overflow and the oversized PDP Add-to-Cart bar.**
   Card #87 `BP5q32o4` — https://trello.com/c/BP5q32o4
   Pure CSS. Run **after** UIC-1 (UIC-1 deletes SCSS blocks in `discover.scss`,
   `product-card.scss`, `product-detail.scss`). *Gated on open question 4 (blast radius of
   the global `border-box` reset).*

Serial order UIC-1 → UIC-2 → UIC-3 avoids conflicts. **Superseded at implementation time:**
all three shipped on one branch and one PR — see Implementation Notes below for why.
## Implementation Notes

**Shipped as ONE bundled branch and PR** — `feat/IP59Crue-storefront-ui-cleanup`, six commits,
2026-08-12. The "one branch per card" line above is superseded: UIC-1 deletes snackbar call
sites and SCSS blocks that UIC-2 and UIC-3 then edit, all three converge on `styles.scss`,
`discover.*`, `shop.*` and `product-detail.*`, and UIC-2's placement AC is judged against
UIC-3's sticky-bar geometry. Bundling is an owner-approved exception to one-card-one-branch,
same precedent as PRs #26/#27/#28. Executed sequentially, one commit per slice.

**Contract impact: none.** No `@hb/shared`, DTO, endpoint, migration, or money/inventory/
order-state change anywhere. `apps/web` only.

### UIC-1 — dead nav + the "SME Verified" concept (`94c7b8f`)

Removed the "SME Directory" / "Logistics" nav buttons, the `/shop` "SME Verification" hero
CTA and `onHeroSmeVerification()`, the "SME Verification" and "Logistics Partnership" footer
links, the SME VERIFIED badge on product cards, the PDP hero badge, the "Verified SME" badge
on the public vendor profile, the vendor-showcase check plus `isVerified()`, and the
`/discover` toggle with `smeOnly` / `filteredProducts` / `onSmeToggle`. Dropped "verified"
from two `shop.html` copy strings (keeping "SME") and the unreferenced `--hb-sme-badge-bg`.

**Behaviour change, owner-confirmed (OQ1).** The toggle was keyed on `product.vendor`, not on
verification status, and defaulted to `true` — so `/discover` was hiding platform-fulfilled
listings. Customers now see platform + vendor listings by default for the first time.
`resultCount` and the grid read `products()` directly. Verified live: `/discover` renders all
10 seeded products including vendor-less ones.

### UIC-2 — snackbar centralisation (`28b5153`, `96a2a3d`)

New `apps/web/src/app/core/notifications/notification.service.ts` (`providedIn: 'root'`) owns
the default `MatSnackBarConfig` — bottom-center, 4000ms, 5000ms for errors — and exposes
`success` / `info` / `error`, each returning the `MatSnackBarRef` so `.onAction()` chaining
("View cart") keeps working. No inline-config override is exposed; that is what enforces the
centralisation. All 19 `.open()` call sites across 9 components migrated, message text
byte-identical. Severity re-mapped where the old site was wrong: add-to-cart and
add-to-wishlist confirmations info → success, cart/wishlist failure paths info → error.

**Real bug the cards did not describe — the reason the toast was unreadable.** `styles.scss`
set `--mdc-snackbar-container-color` and `--mdc-snackbar-supporting-text-color`. Angular
Material 21 reads `--mat-snack-bar-container-color` / `--mat-snack-bar-supporting-text-color`
(see `.mat-mdc-snack-bar-container .mat-mdc-snackbar-surface` in the snack-bar component
styles). Both declarations were inert, so the surface fell through to
`var(--mat-sys-inverse-surface)` — undefined, because the app loads no Material theme — and
rendered **fully transparent with inherited dark body text**. Moving the toast to the bottom
alone would not have fixed it. Only `--mat-snack-bar-button-color` was already correct, which
is why the action label was the one part that ever picked up colour. The same applied to
`--mat-snack-bar-container-shape` and the supporting-text tokens (0px radius, inherited
16px/normal), fixed alongside: now 4px with Inter 14px/20px/400.

Contrast measured off live computed styles (light theme; the app has no dark theme), all
above WCAG AA 4.5:1:

| Variant | Text on container | Action button |
|---|---|---|
| success | `#ffffff` on `#015300` — **9.38:1** | `#98f982` — **7.24:1** |
| info | `#f3f0ef` on `#313030` — **11.60:1** | `#7ddc69` — **7.71:1** |
| error | `#ffffff` on `#ba1a1a` — **6.46:1** | `#ffdad6` — **5.00:1** |

`.cdk-overlay-container` is z-index 1000, above `.pdp__sticky-bar` (40) and `.radial-nav`
(60), so the toast is never occluded.

### UIC-3 — box model + PDP sticky bar (`7419bd5`)

Global `box-sizing: border-box` reset in `styles.scss` (`html` sets it, `*, *::before,
*::after` inherit), plus `max-width: 1280px` and `left: 50%` / `translateX(-50%)` centring on
`.pdp__sticky-bar`, and `flex: 0 1 400px` on `.pdp__add-to-cart-btn` from 768px up (mobile
keeps `flex: 1` — the bar is a mobile pattern by design). No `overflow-x: hidden` anywhere.
The nine pre-existing local `box-sizing` declarations were deliberately left in place.

**The diagnosis was measured on both sides of the change, not assumed.** At a 1280px viewport
`/shop` and `/discover` reported `scrollWidth` 1345 vs `clientWidth` 1265 before the reset —
overflow of exactly the 80px of horizontal padding the root-cause section predicted — and 0
after. Zero overflow after the fix at 1280 / 1366 / 1440 / 1920 on `/shop`, `/discover`,
`/products/:id`, a public vendor profile, `/cart`, `/checkout`, `/wishlist`, `/login`,
`/register`, `/forgot-password`, `/reset-password`, and the admin portal (`/admin/dashboard`,
`/orders`, `/users`, `/catalog`, `/vendors`, `/logs`, `/search-synonyms`).

**Blast radius (OQ4), all benign.** Admin/profile/vendor shell sidebars went 265px → 240px —
they now honour their declared `width: 240px` instead of adding padding and border on top; 9
admin nav items measured, zero clipped, page overflow 0. `.auth-card` 514 → 448px, admin row
buttons −32px, synonym rows −8px. Every one of these is a previously-overflowing box now
fitting its container.

### Review round

code-reviewer verdict **FIX-FIRST**. One blocking finding, fixed in `d7e53a3`: the PDP vendor
card still rendered a filled `verified` glyph gated on `@if (product.vendor)` — the identical
redundant assertion, missed by the UIC-1 sweep. A customer would have seen a verified check on
`/products/:id` and no mark for the same vendor on `/vendors/:id`. Removed with
`.pdp__vendor-verified` and the now-single-child `.pdp__vendor-name-row` wrapper; the spec that
claimed to assert the badge was retitled and given real negative assertions. Same commit
removed two orphaned SCSS rules whose only consumers UIC-1 deleted (`.nav-bar__link--btn`,
`.btn--ghost`).

### Test / build state

`npm run test -w @hb/web` — **63 files / 782 tests green**, zero skipped. `npm run build` clean
apart from two pre-existing SCSS budget warnings (`admin-catalog.scss`, vendor-portal
`vendor-profile.scss`), neither touched here. No API tests run — no API code changed. Evidence
recompiled in `ae7ff40`.

### Follow-ups (not in this batch)

- **Mobile horizontal overflow is still real and is a different bug.** `/discover` overflows
  ~200px at 375px and ~479px at 768px. Measured both sides of the change: 204px before, 200px
  after — the reset shrinks it slightly but does not cause it. Root cause is the nav-bar
  actions row and the discover product grid's intrinsic widths, not the box model. These
  cards' ACs cover ≥ 1280px only. Needs its own card.
- **The design source is now divergent.** `docs/design/claude-design/` exports still carry
  "SME Verified" and "SME Directory". Claude Design is the declared source of truth, so the
  next screen built from an export could reintroduce the badge. Worth a design-sync pass.
- OQ5 (the remaining dead `href="#"` footer links) is untouched, as scoped.
