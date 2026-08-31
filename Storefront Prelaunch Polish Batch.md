---
type: implementation
tags:
  - storefront
  - ui
  - prelaunch
---

# Storefront Prelaunch Polish Batch

Status: **shipped 2026-08-31**, PR open, awaiting human merge.
Related: [[Storefront UI Cleanup Batch]] · [[Public Storefront & SSR]] · [[HB Domain Model]]

## Problem

Six customer-facing cosmetic and copy corrections for the pre-launch storefront. All are pure UI/content
(no API, no schema, no money/inventory/order logic), and four converge on the same two files (`footer.html`/`footer.scss`,
`nav-bar.scss`), so they are bundled into one branch rather than six separate PRs.

1. **Brand anchors on auth pages and sidebars link to themselves or their own dashboards.** Login, Signup,
   Admin shell, Vendor shell all render a logo/wordmark with an href pointing back to the current page
   or to the portal itself — not to the customer storefront.
2. **Logo reads too small.** The nav-bar rendered `hb-logo.png` at 32px and the footer at 28px. The
   shipped asset is 96×96 *precisely because* of those two sizes — 3× coverage for DPR 3 screens — so
   enlarging the render without regenerating the asset would have traded one defect for a soft logo.
   That constraint, not the card, is what forced the pipeline change.
3. **Footer social links are placeholder glyphs with no destinations.** Three Material icon spans (`public`,
   `shield`, `payments`) styled with a hover transform look clickable but have no href; the text "social
   links" is misleading.
4. **Mobile navbar overflows horizontally by 158px.** Measured at 375px viewport width.
5. **Search placeholder copy is vague and outdated.** Home page reads "Search SME products…"; after UIC-1
   removed the "Verified SME Vendors" toggle, `/discover` now shows all products (platform + vendor), so
   the placeholder no longer accurately describes what a customer can search.
6. **Contact page copy suggests only quote/import inquiries.** Page title ("Get a Quote or Request an Import"),
   form labels, and section headings all frame the page as vendor/import-specific, though the actual
   form's `orderType` select includes "general inquiry". This constrains perceived purpose.

## Business rule: storefront pre-launch gating

No stored business rule change — this is cosmetic correction driven by the **owner-approved pre-launch sweep**
before the first external announcement. Decisions were made by the owner during implementation (search placeholder
wording, social media link choice, logo sizing strategy).

## Scope

### 1 — Brand anchor repointing (auth + sidebars)

- `apps/web/src/app/auth/login/login.html` — logo's href currently `[routerLink]="undefined"` (navigates nowhere).
  Point to `/shop`.
- `apps/web/src/app/auth/register/register.html` — same.
- `apps/web/src/app/layout/admin-shell/admin-shell.html` — wordmark links to `/admin/dashboard`. Point to `/shop`.
- `apps/web/src/app/layout/vendor-shell/vendor-shell.html` — wordmark links to `/vendor/dashboard`. Point to `/shop`.

### 2 — Logo asset regeneration and sizing

- `apps/web/src/assets/images/hb-logo.png` — regenerate from 96×96 → **144×144** (from the 2000×2000
  source via `scripts/optimize-brand-assets.py`, not upscaling the shipped asset).
- `apps/web/src/app/layout/nav-bar/nav-bar.html` — `.nav-bar__brand` image size 32 → **44px**.
- `apps/web/src/app/layout/footer/footer.html` — footer logo 28 → **40px**.

### 3 — Footer social links (replacement + URLs)

- `apps/web/src/app/layout/footer/footer.html` — replace three bare Material icon spans with inline SVG links
  to WhatsApp, Instagram, TikTok (WhatsApp is the only messaging link; Facebook/YouTube were considered
  but no page exists).
- New export `SOCIAL_LINKS` in a new utility file `apps/web/src/app/shared/utilities/constants.ts` with
  three objects: `{ platform, label, icon, url }`.

### 4 — Mobile navbar overflow

Measure and fix the 158px horizontal overflow at 375px viewport.

### 5 — Search placeholder copy

- `apps/web/src/app/features/shop/shop.html` line ~29 — "Search SME products…" → **"Shop our latest products"**.
- `apps/web/src/app/features/discover/discover.html` line ~36 — same change.

### 6 — Contact page copy broadening

- `apps/web/src/app/features/contact/contact.ts` — `PAGE_TITLE` / `PAGE_DESCRIPTION` currently "Get a Quote
  or Request an Import"; broaden to describe general inquiries and vendor partnerships.
- `apps/web/src/app/features/contact/contact.html` — section headings and form field labels should not frame
  the page as import-only.

## Data / contract impact

**None.** No `@hb/shared` change, no DTO, no endpoint, no migration, no money/inventory/order-state logic.
`apps/web` only.

## Out of scope

- `product-detail.html` encoding corruption on two "Loading…" strings — **deliberately deferred**: the file
  has a UTF-8 / CP1252 mojibake bug (bytes `C3 A2 E2 82 AC C2 A6` rendering as `loadingâ€¦`) that needs
  its own diagnosis and card.
- `profile-shell` wordmark-links-to-its-own-dashboard pattern — not named in card 3QJYmybN.
- Contact form `orderType` select labels and `PAGE_DESCRIPTION` mismatch — a "support" option would require
  an `@hb/shared` enum + DTO + column change, a real card, not a copy-only fix. Copy alone goes here.
- Design system sync — the Claude Design source still carries "SME Verified" and "SME Directory" exports;
  that is a separate sync cycle.

## Open questions (resolved before /ship-card)

1. **Search placeholder wording** — "Shop our latest products" (owner choice, confirmed during implementation).
2. **Social media channels** — WhatsApp + Instagram + TikTok only (Facebook/YouTube have no active pages).
3. **Logo sizing strategy** — regenerate from source at 144×96 rather than upscale the 96×96 shipped asset
   (preserves sharpness; the 96px asset was generated precisely to cover DPR-3 renders of 32px/28px).
4. **Admin/Vendor wordmark destination** — `/shop`, not their respective dashboards (owner choice).

## Vertical slices (→ Trello cards)

1. **3QJYmybN — Fix storefront branding links + increase logo size**
   Brand anchor repointing (Login/Signup/Admin/Vendor shells) + logo asset regeneration + sizing.

2. **wLQHce2J — Auth pages: logo/name design polish + Support → Contact link**
   Login gets the real logo treatment (currently uses a generic Material glyph in the logo position).
   Register logo polish (matching Login). Both auth pages gain a `/contact` link (replacing the "Support"
   notifyComingSoon stub).

3. **6nnBD4hS — Footer icons: add social links, remove unused**
   Replace placeholder Material glyphs with real WhatsApp/Instagram/TikTok SVG links.

4. **4Rugdi1d — Mobile: fix navbar overflow + footer link text size**
   Diagnose and fix the 158px horizontal scroll at 375px. Enlarge footer link text (28 → 40px, matching
   the logo size increase).

5. **6p4mjTFS — Fix search placeholder copy (Home + Search page)**
   Wording from "Search SME products…" to "Shop our latest products" on both `/shop` and `/discover`.

6. **wrpd9lGc — Contact page: broaden messaging beyond inquiries**
   Copy and layout refresh to frame the page as vendor partnerships + general inquiries, not just import requests.

## Implementation Notes

**Shipped as ONE bundled branch and PR** — `feat/3QJYmybN-storefront-prelaunch-polish`, 7 commits,
2026-08-31. One card per commit, same owner-approved bundling precedent as the Storefront UI Cleanup Batch
(PRs #26/#27/#28). Four of the six cards converge on `footer.html`/`footer.scss` (three) and `nav-bar.scss`
(two), and the logo regeneration feeds into both bundled sizing changes, so the dependency graph is too dense
for serial one-card-one-branch execution.

**Contract impact: none.** No `@hb/shared`, DTO, endpoint, migration, or money/inventory/order-state change.
`apps/web` only.

### 3QJYmybN — brand links + logo sizing (`bbf8105`)

Brand anchors on Login/Signup/Admin/Vendor shells repointed to `/shop` (Login was `undefined`, portals linked
to their own dashboards). Logo asset regenerated 96×96 → 144×144 from the 2000×2000 source via
`scripts/optimize-brand-assets.py` (bumping `LOGO_SIZE` constant), preserving sharpness at the new 44px (nav)
and 40px (footer) render widths. Pre-existing 32px render at DPR 3 was covered by the 96px asset; 44px at
DPR 3 = 132px on-screen, so 144px is the minimal power-of-2 asset size.

### wLQHce2J — auth page polish (`bf6074e`)

Login now renders the real `hb-logo.png` in the logo position (previously displayed a generic Material `hub`
glyph). Register kept the glyph but both now share a matched `.auth__brand` treatment with Login's real mark.
"Support" button (a `notifyComingSoon()` stub) replaced with a real `/contact` link; Register gained the
matching link. Both links tested for reachability.

### 6nnBD4hS — footer social links (`76e00c1`)

Three bare Material icon spans (`public`, `shield`, `payments`) with no href replaced with inline SVG links
to WhatsApp, Instagram, TikTok. New utility export `SOCIAL_LINKS` holds the URLs and platform labels; each
link opens in a new tab (`target="_blank"`, `rel="noopener noreferrer"`).

### 4Rugdi1d — mobile overflow fix + footer link sizing (`a2173dc`)

**Diagnosis: two independent causes of 158px overflow at 375px.**

1. **Nav-bar action controls had no mobile breakpoint.** Sell button, search icon, utility controls all rendered
   side-by-side with no collapse. Fixed by restoring them only at **1280px** (not 768px as set in the previous
   batch). Search icon hidden below 768px. Brand scaled 24 → 18px below 768px and to the mark alone below 360px,
   with `nowrap` and `flex-shrink: 0` to stop the wordmark breaking into two lines.

2. **Vendor section overflow was not the nav.** `.vendors-section { width: 100%; margin: 0 16px; }` in `shop.scss`
   added padding that overflowed a 375px container by exactly one 16px margin. Changed to `width: auto`.

**The 1280px breakpoint is load-bearing for another reason:** during code review, when the breakpoint was first set
to 1240px (from arithmetic), a measurement sweep at 320/360/375/390/414/430/768/1024/1085/1280/1440 passed but did
not include 1240px itself. At 1240px viewport, the nav children + gaps measure ~1218px against a 1200px content box
(once `.nav-bar__inner` hits its `max-width: 1280px`), yielding ~32px of real overflow. **Lesson: breakpoint sweeps
must include the breakpoint values themselves, not just the device widths either side.**

Footer link text enlarged 28 → 40px to match the logo sizing change (all three — `.footer__logo` `width`, `.footer__logo` `height`,
`.footer__social-link` `font-size`).

### 6p4mjTFS — search placeholder copy (`bf6074e` combined with 4Rugdi1d)

Changed "Search SME products…" to **"Shop our latest products"** on both `/shop` and `/discover`. The old wording
no longer accurately described the catalogue after the Storefront UI Cleanup Batch (UIC-1) removed the "Verified SME
Vendors" toggle, which was hiding platform-fulfilled listings. Customers now see all products by default.

### wrpd9lGc — contact page copy broadening (`bf6074e` combined with others)

`PAGE_TITLE` / `PAGE_DESCRIPTION` and section headings reworded from "Get a Quote or Request an Import" to frame
the page as a general vendor partnership and inquiry portal, not import-specific. Form's `orderType` select already
included "general inquiry", so the copy now matches the form's capability.

### Two findings worth recording as knowledge

**1. `discover.html` was shipping corrupted text (mojibake) to customers.** The file was UTF-8 decoded as CP1252 and
re-saved. The search placeholder's ellipsis was stored as bytes `C3 A2 E2 82 AC C2 A6` and rendered live as
`productsâ€¦`; "Loading products" and three comment separators were mangled the same way; the file also carried a BOM.
Repaired here because the corruption sat inside the exact string card 6p4mjTFS replaces. **`product-detail.html` still
has the identical corruption on two "Loading…" strings — deliberately out of scope, needs its own card.**

**2. The mobile overflow had two independent causes, and the nav one was mis-scoped by the Storefront UI Cleanup Batch.**
The previous batch set a 768px breakpoint for the nav Sell button to disappear. But this measured only devices at 768px
and above; it did not verify the exact breakpoint value itself. At 1240px, the nav overflowed. At 1280px (where the
inner container's `max-width` hits), children + gaps = 1218px against a 1200px content box — 32px of real overflow.
Correcting to 1280px is correct; the lesson is that a measure-and-fix sweep must include **all the breakpoint values,
not just the device widths either side of them.**

### Test / build state

- `npm run test -w @hb/web` — **80 files / 1073 tests green** (up from 1058 pre-batch; +15 new for brand links, footer
  social links, and search placeholder copy).
- `npm run build` clean apart from four pre-existing SCSS budget warnings (`admin-catalog`, `product-detail`, vendor
  `vendor-profile`, `admin-orders`) — none touched in this batch.
- No API tests run: no API code changed.
- Card 4Rugdi1d's responsive behaviour (viewport widths, breakpoint thresholds) is deliberately **not** unit-tested —
  jsdom does no layout and never evaluates media queries, so such a test would assert nothing real. Verified live across
  **320, 360, 375, 390, 414, 430, 768, 1024, 1085, 1240, 1280, 1300, 1366, 1400, 1440** viewports on dev server,
  measuring zero horizontal overflow on `/shop`, `/discover`, `/contact`, `/login`, `/register`, `/admin/*`, `/vendor/*`.

### Verification detail on mobile overflow fix

- **Before:** `/shop` at 375px reported `scrollWidth - clientWidth = 158px`; `/discover` at 375px reported `+200px`;
  `/contact` at 375px reported `+104px`.
- **After:** all pages at all measured widths report zero overflow except the documented out-of-scope cases (nav actions
  and product grid intrinsic width on mobile, which are a different bug).
- Sweep included 1240px explicitly (the breakpoint value that failed the previous batch's sweep).

### Follow-ups (not in this batch)

- **`product-detail.html` mojibake on two "Loading…" strings** — UTF-8/CP1252 decoding corruption, same root cause as
  the `discover.html` fix. Needs its own diagnosis and card.
- **`profile-shell` wordmark pattern** — the user profile shell has the same wordmark-to-dashboard link pattern as
  Admin/Vendor; not named in card 3QJYmybN, left alone. May need a follow-up.
- **Contact page `PAGE_DESCRIPTION` / `orderType` label mismatch** — copy now frames the page broadly, but `PAGE_DESCRIPTION`
  still says "Get a Quote or Request an Import", and select labels still list only quote/import/inquiry. A separate card
  touching only `contact.ts` would close the copy gap, but adding a "support" option would require `@hb/shared` enum +
  DTO + column change (real card, not copy-only).
- **Mobile nav and grid intrinsic width overflow** — `/discover` still overflows ~200px at 375px (nav actions + product grid).
  Root cause is not the box model (fixed) but the action controls and grid column count. Cards 3QJYmybN / 4Rugdi1d scope
  only ≥ 1280px; this needs its own card.
