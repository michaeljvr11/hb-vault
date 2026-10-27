---
type: spec
tags:
  - qa
  - storefront
  - portals
  - checkout
---

# Site QA Findings Batch

Status: **specced 2026-10-27**, cards QA-1 … QA-21 in Trello "To Do". Not started.
Related: [[Storefront UI Cleanup Batch]] · [[Storefront Prelaunch Polish Batch]] · [[Money & Currency Rules]] · [[Vendor Earnings & Commission]] · [[Vendor & Admin Portals]] · [[Customer Profile]] · [[Configurable Shipping Fee]] · [[Shop Home Sections]] · [[Product Discounts & Sale Pricing]] · [[Legal & Compliance Readiness]] · [[Transactional Email & Order Notifications]] · [[Auth & Roles]]

## Problem

21 findings from a manual QA pass over the site (storefront, checkout, customer profile,
vendor portal, admin portal, legal pages). Each was checked against the code on
`main` @ `9eb9c68` before it was turned into a card. Most are cosmetic. Four touch
the API: QA-13 (discount savings, money), QA-17 (public shipping fee read),
QA-20 (deactivation reason + email, schema) and QA-21 (commission rate validation, money).

One card per finding, so each one can be tracked and closed on its own. They are small,
so `/ship-batch` should bundle them. The suggested batches are at the bottom.

## Findings, root cause, and decision

### Storefront

**QA-1: Explore categories tiles: remove the arrow, centre the text.** The `/shop`
"Explore categories" tile has an `arrow_forward` chevron (`shop.html` ~196,
`.category-card__chevron` in `shop.scss` ~411, shown from `md` up). The tile is
left-aligned (`text-align: left`, icon beside the text). *Assumption:* "categories section"
means these tiles, not the header category bar.

**QA-2: Icons show as words on first load.** Material Symbols is self-hosted
(`src/fonts.css`) as the 4.0 MB `full` variable woff2 with `font-display: swap` and no
preload. Until it arrives, every icon renders its ligature name as text ("shopping_cart",
"expand_more"), which reflows the layout. Constraint from `apps/web/CLAUDE.md`: the font
**must keep all four axes** (FILL is toggled in many places), so do not switch to a static
or FILL-less build. Fix direction: `font-display: block` for the icon face (hidden briefly,
never shown as words), preload the woff2, and subset the **glyphs** to the icon names the
app uses while keeping every axis. The CSP and self-hosting rules stay as they are.

**QA-3: The "All categories" flyout opens, closes and reopens.** *Confirmed with owner:*
hover and click fight each other. `category-nav.ts`: hovering the trigger opens the flyout
after `HOVER_INTENT_MS` (400 ms), and a click calls `toggle()`. So a shopper who hovers and
then clicks (the usual action) closes the panel the hover just opened, and the next
interaction opens it again. Fix: once a hover has opened the panel, a click on the trigger
keeps it open (or a click within a short window of a hover-open is ignored). Keep the
keyboard, Escape, focus trap and reduced-motion behaviour as they are.

**QA-9: The nav bar changes width when you switch ZAR/NAD.** The currency button label
(`nav-bar.html` ~67) is proportional Inter text, so "ZAR" and "NAD" have different widths
and the flex row moves. Fix: give the label a fixed width (tabular numerals, `min-width`
in `ch`, or a fixed button width).

**QA-10: A new page should start at the top.** `app.config.ts` calls
`provideRouter(routes)` without `withInMemoryScrolling`, so the browser keeps the old
scroll position when you navigate. Fix: `scrollPositionRestoration: 'enabled'` (top on a
new navigation, the old position on back/forward). *Decision:* a navigation that only
changes query parameters on the same path (for example `/discover` filters and paging)
should **not** jump to the top, so filters stay usable. Paging may scroll to the top of
the results list.

**QA-11: The wishlist button changes size when clicked.** On the PDP the label swaps
"Save to wishlist" → "Saved" (`product-detail.html` ~197), so the button shrinks and the
`flex: 1` Add to Cart grows. Fix: a stable width for the button whichever label shows.
The product-card heart "pop" (`product-card-pop`, scale 1.22) is intended and uses only
`transform`, so it causes no reflow. Keep it unless QA shows the card heart is the one
that moves.

**QA-18: Shop shelves need an edge fade and arrows.** `product-shelf` scrolls sideways with
the scrollbar hidden (`product-shelf.scss` ~42) and gives no sign there is more content.
Fix: a fade mask on any edge with more content, plus previous/next arrow buttons on
pointer devices (`md`+) that scroll one page of cards. The arrows hide or disable at each
end. Touch keeps swiping. If you use scroll-driven CSS, verify it with a headless probe: the
desktop app's browser pane reports scroll-timeline progress wrongly. Driving the state from
JS scroll events avoids the problem.

### Checkout and currency

**QA-12: Saved addresses never reach checkout.** The address book exists
(`AddressesService`, profile → Addresses), but `checkout.ts` builds an empty form that
defaults to NA and never reads the saved addresses. Fix: a signed-in shopper with saved
addresses chooses one (most recent preselected, matching the `DisplayCurrencyService` rule
"most recent address = default"), and it fills the form. "Use a new address" clears it.
No contract change (`AddressDto` → `CreateAddressRequest` map 1:1). Out of scope: saving a
new checkout address back to the book, and a default-address flag.

**QA-13: Checkout should show the total discount.** `CartItemDto.unitPrice` is the
*effective* (discounted) price (`cart.service.ts` ~187, `resolveEffectivePrice`). The
undiscounted price never reaches the client, so the UI cannot show savings without doing
money arithmetic it is not allowed to do. **Contract change:**
`CartItemDto.listUnitPrice` (undiscounted, present only while a discount is active) and
`CartTotalDto.discountTotal` (per currency, server-computed in integer cents, never summed
across currencies). Checkout and cart then show "You save N$X" as a summary row. This is
money logic, so it needs unit tests (mixed discounted and undiscounted lines, rounding, one
total per currency, an expired discount).

**QA-14: Tidy the checkout. The trust content should expand on click.** Under the payment
method the checkout stacks a "How payment works" list, an inline `<app-trust-banner>`, the
logistics card and a "1:1 Regional Parity Guaranteed" line. Fix: fold "How payment works"
and the trust items into one disclosure (closed by default, keyboard accessible, a one-line
summary when closed). Keep the logistics card and the totals as they are. Related open card
**SVO-9** (`61cBONbQ`, the security block names no payment provider) edits the same block,
so ship them together or one after the other.

**QA-15: It isn't clear which currency you pay in.** *Owner decision 2026-10-27: keep the rule
and explain it.* The rule ([[Money & Currency Rules]]): an order is charged in the product's
**listed** currency. The nav switch is **display only** and converts browse prices at the
peg. `settlementCurrencyNote` already shows a note on cart and checkout, but only when the
display currency differs from the settlement currency. The PDP shows converted prices with
no hint. Fix (UI and copy only): label the switch as a display preference (for example
"Show prices in"), say "Charged in ZAR" on the PDP when the display currency differs from the
listing currency, and always show "You pay in <currency>" on cart and checkout with a short
explanation. No change to the money rules.

### Legal pages

**QA-16: The WhatsApp link on the returns page is broken.** `returns-policy.html` ~17 and
`terms-of-service.html` ~21 print `wa.me/264…` as plain text, with no `<a>`. Fix: a real
`https://wa.me/<E164>` link built from `CONTACT_DETAILS` (add a plain `whatsappUrl` to
`site.constants.ts` so the number stays in one place), `rel="noopener"`.

**QA-17: The shipping policy doesn't show the fee.** `shipping-policy.html` ~29 still has
`[SHIPPING FEE AMOUNT]`. The fees are admin data ([[Configurable Shipping Fee]]), but
`GET /shipping-fee/current` needs a signed-in user and works one route at a time, and
`/legal/shipping` is `RenderMode.Prerender`. Fix: a new **public** read endpoint listing
the fee set in force (route + currency + amount, no per-product overrides, no audit
fields), fetched in the browser only, with a fallback ("shown at checkout") on error.
Hard-coding the amount would go stale the next time an admin changes it. Mention that some
products carry their own fee. Not gated on LC-1: this is admin data, not legal-entity text.

### Customer profile

**QA-4: Change password: enter the new password twice.** `profile-details.html` has
Current and New fields only. Add a "Confirm new password" field with a group validator.
Reuse `passwordsMatch` from `auth/reset-password/reset-password.ts` (move it to
`shared/validators`). Show the mismatch error on the confirm field and disable submit
until they match. *Decision:* **keep "Current password"**. Removing it would let anyone
with an unlocked session take over the account. The confirm field is client-side only, so
there is no contract change (`ChangePasswordRequest` stays `{ currentPassword, newPassword }`).

### Vendor and admin portals

**QA-5: Spacing between quick links and Analytics.** On the vendor dashboard,
`.quick-links` (Manage Products / View Orders) has no bottom margin, so the Analytics
`.section-card` sits flush against it (`vendor-dashboard.scss` ~177). The admin dashboard
already has `margin-top: 32px` on `.analytics-section`. Check it at the same widths, but
the gap is the vendor one.

**QA-6: Add/Edit Product as a centred modal.** `vendor-products.html` ~175 opens the
product form in a right-hand `.drawer`. The admin catalogue (`admin-catalog.html`) uses
the same drawer pattern. Fix: a centred modal dialog (max-width, scrolling body, sticky
footer actions, full-screen sheet below `md`), keeping the focus trap, Escape, the backdrop
and the unsaved-changes behaviour. *Assumption:* the admin catalogue gets the same change
so the two portals match. Drop it from the card if the owner wants vendor only.

**QA-7: Tidy the custom date pickers.** "Custom start / Custom end" in the shared
`earnings-range-selector` (vendor and admin earnings) are outline `mat-form-field`s next to
pill tabs. They have different heights and baselines, a loose "–" separator, and an inline
error that pushes the layout. Fix: align with the tabs, a compact density, one range field
(`mat-date-range-input`) or a matched pair, and an error that does not shift the row.
Native `type="date"` inputs in admin logs, tracking and commission are out of scope.

**QA-8: The settlement period table wraps and glitches.** `vendor-earnings.scss` ~224 puts
`display: flex` **directly on a `<td>`** (`.settlement-money-cell`), which takes it out of
table-cell layout: the row's borders and heights stop lining up. The period cell
("1 Oct 2026 – 14 Oct 2026") also wraps. Fix: put the flex column on an inner wrapper,
`white-space: nowrap` on the period, and check the table at narrow widths (horizontal
scroll inside `.table-card`, not page overflow).

**QA-19: Vendor application answers aren't visible on the admin side.** *Confirmed with owner:*
the vendors checked were old or hand-inserted, so they had no application answers.
`admin-vendors.html` wraps each answer in `@if`, so a missing answer **disappears** and
admin cannot tell "not provided" from "not built". Fix: always show the application
section with every field. Show "Not provided" for an empty one, and a note when the vendor
applied before the form had those fields. Make sure the dev/demo seed vendors carry full
application answers.

**QA-20: Deactivating a user needs a reason and an email.** `PATCH /admin/users/:id/active`
takes `{ isActive }` only (`admin.service.ts` ~80) and only writes an audit row. *Owner
decision 2026-10-27:* the admin must choose a **reason category** (Fraud / suspicious
activity, Policy breach, User request, Other) and can add an internal note. The user gets
an email that names the category, never the internal note, plus a support contact.
Reactivation sends its own email. Contract: `SetUserActiveRequest` gains `reason?`
(enum, required when `isActive=false`) and `note?`. Store the reason and note on the user
(migration) and in the audit metadata. Show the latest reason in the admin user detail.
Mail goes through `MailService`. A send failure must not roll back the deactivation (log
and continue). Out of scope: a self-service appeal flow, and the vendor suspension flow
(that is a separate lifecycle, see [[Listing Types & Vendor Rules]]).

**QA-21: Commission rate validation.** `CommissionRateService.create` only checks
`effectiveFrom > latest.effectiveFrom`. If the latest rate started two months ago, an admin
can schedule a new one from last week. Order items snapshot `commissionRatePercent` at
creation, so no order changes, but the history would then say a rate applied to orders
that were really charged the old one. Nothing stops scheduling the **same** percent as
the rate before it either. Rules:
1. `effectiveFrom` may not be in the past (server clock, with a small skew allowance of
   about 5 minutes; omitting it still means "now").
2. `ratePercent` must differ from the rate in force immediately before `effectiveFrom`,
   which is the latest row given rule 1 and the strictly-increasing rule.

Both are 400/409 errors with clear messages, unit-tested in
`commission-rate.service.spec.ts`. The admin form (`admin-commission`) mirrors them on the
client: `min` on the datetime input and a "same as current" error. No schema change.

## `@hb/shared` contract impact

| Card | Change |
|---|---|
| QA-13 | `CartItemDto.listUnitPrice?: number`, `CartTotalDto.discountTotal: number` |
| QA-17 | new `PublicShippingFeeDto` (or reuse `CurrentShippingFeeDto[]`) for the public endpoint |
| QA-20 | `SetUserActiveRequest.reason?`/`note?`, new `DeactivationReason` enum, `AdminUserDto.deactivationReason?`/`deactivatedAt?` |
| QA-21 | none (validation only) |

Migrations: QA-20 only (user deactivation reason, note and timestamp).

## Out of scope

- Changing which currency an order settles in (QA-15 is explanation only).
- Saving a new checkout address to the address book; a default-address flag.
- Restyling native date inputs outside the earnings range selector.
- A self-service appeal flow for deactivated users.

## Suggested `/ship-batch` groupings

1. **Storefront polish:** QA-1, QA-3, QA-9, QA-10, QA-11, QA-18 (web only)
2. **Icons:** QA-2 on its own (build and font pipeline, check it in prod-mode SSR)
3. **Checkout:** QA-12, QA-14 (+ SVO-9), QA-15, then QA-13 (contract + API + tests)
4. **Legal pages:** QA-16, QA-17 (QA-17 adds a public endpoint)
5. **Portal polish:** QA-4, QA-5, QA-6, QA-7, QA-8, QA-19 (web only)
6. **Admin rules:** QA-20, QA-21 (API + migration + mail + tests)

## Cards

| Code | Card | Title |
|---|---|---|
| QA-1 | `yGRba8mk` | Explore categories tiles: remove the arrow and centre the text |
| QA-2 | `eFktpP5m` | Icons show as words on first load: fix the icon font loading |
| QA-3 | `NXIBdJ7c` | "All categories" flyout opens, closes and reopens |
| QA-4 | `ZGEtbGcG` | Change password: confirm the new password |
| QA-5 | `cPWBQEsU` | Vendor dashboard: spacing between the quick links and Analytics |
| QA-6 | `gqE0a6e5` | Add/Edit Product as a centred modal |
| QA-7 | `E9VhpMCi` | Tidy the custom date range pickers |
| QA-8 | `7qpJ5S5g` | Settlement periods table wrapping and misaligned rows |
| QA-9 | `WR6Extzq` | Nav bar shifts width when switching ZAR/NAD |
| QA-10 | `m5WUCkuL` | Every new page opens scrolled to the top |
| QA-11 | `cPMDYtHG` | Wishlist button changes size when clicked |
| QA-12 | `syUWbsqu` | Checkout: choose from saved addresses |
| QA-13 | `I6pewX9r` | Show the total discount on checkout and cart |
| QA-14 | `8wFo0GA9` | Tidy the checkout: one expandable payment/trust section |
| QA-15 | `YgW3VC7n` | Make it clear which currency the shopper pays in |
| QA-16 | `pf0T6ggh` | WhatsApp link on returns/terms is not a link |
| QA-17 | `FH1uClnj` | Shipping policy shows live fee amounts |
| QA-18 | `ctR4ktxT` | Shop shelves: edge fade and arrows |
| QA-19 | `ZwHABBKv` | Admin vendor detail: show every application answer |
| QA-20 | `EeGjozdz` | User deactivation: reason + email |
| QA-21 | `FQUHQkzC` | Commission rate: reject past dates and an unchanged rate |
