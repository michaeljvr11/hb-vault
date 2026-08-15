# Landing Site Migration

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Status: **decisions confirmed 2026-08-13** (see Decisions below) — ready for `/ship-card`.
Related: [[Public Storefront & SSR]] · [[Auth & Roles]] · [[HB Domain Model]] · [[Transactional Email & Order Notifications]] · [[Cross-Border & Customs]] · [[Storefront UI Cleanup Batch]]

## Problem

The live business front today is a **separate** Angular 21 project, `hb-landing`
(`C:\Users\michael.jvanrensburg\vscode\HB Repo\hb-landing`, not tracked in this monorepo,
deployed at hb-ecommerce.com). It carried the business while `apps/web` was being built.
`apps/web` now has a storefront, cart, checkout, vendor and admin portals — but **no About,
Services or Contact route at all** (confirmed in `apps/web/src/app/app.routes.ts`), no logo,
and no marketing imagery. Meanwhile hb-landing still frames the marketplace as a future thing
("coming soon" language throughout).

This note covers folding the useful parts of hb-landing into `apps/web`: brand assets, the
three marketing pages, and a copy rewrite that stops describing the marketplace as
"coming soon." Retiring the hb-landing repo/deployment itself is **not** in scope — see below.

## Business rules this must honour

**Dual-engine model — the sourcing service is permanent, not a stopgap.** H&B Brain
`07-business-model.md`: Engine 1 **Procurement Service** (H&B buys on the customer's behalf
from Takealot/Amazon/Shein/Temu/individual vendors, briefly owns the goods, resells at a
tiered margin — 20% under R500, 15% R500–1,000, 10% over R1,000; shipping + customs passed
through). Engine 2 **Marketplace** (vendors sell their own goods; **H&B never owns
marketplace inventory**). `05-problem-we-solve.md` is explicit: "the procurement service
continues alongside the marketplace (revenue line + customer acquisition), but H&B's own
vendors are prioritised long-term." `23-decisions-log.md` (2026-07-14) logs this as an active
decision.

So the migrated Services copy must present sourcing as a **standing complementary offering**,
not a teaser and not a legacy service being wound down. The "never owns marketplace
inventory" distinction matters for marketing copy — blurring the two engines blurs the
accounting and investor story too.

**Support promises the Contact page makes on H&B's behalf** (`15-customer-support.md`,
`12-customer-journey.md`): channels are website form + WhatsApp; **acknowledgement within 1
business day**; after-hours covered; English and Afrikaans. Damage claims within 48 hours of
delivery. Any SLA wording on the new Contact page must match these numbers exactly, or state
no number at all.

**Corridor framing** (`02-vision.md`, `06-target-customers.md`): Namibia-first demand,
SA↔Namibia corridor, delivery to all of Namibia from day one (beyond-Windhoek at the
customer's own higher cost). Copy should not promise "worldwide" beyond what the business
model supports (SA and globally-sourced goods *into* Namibia).

## ⚠️ Conflict: "the marketplace is now live"

The request's framing doesn't match H&B Brain's current state, and this is a product/launch
decision, not a copy edit:

- `18-product-roadmap.md`: **Marketplace v1 target is 1 October 2026** (today is 2026-08-13).
  Phase 0 "live today" is still the landing page + WhatsApp/EFT procurement.
- `23-decisions-log.md` (2026-08-11): **2 of 10** founding vendors confirmed. The
  "recruit 10 founding vendors" launch blocker is still open.
- `18-product-roadmap.md` / `13-payments-payouts.md` / `24-risks-assumptions.md` (risk A1):
  **no payment gateway is chosen.** Stitch's Namibian support looks absent; FNB Namibia
  eCommerce Switch and DPO Group are under evaluation.
- `16-marketing-strategy.md`: the "Marketplace Launch Moment" section is flagged
  **no launch plan yet** — waitlist, founding-customer promo, first vendor spotlights are
  explicitly "not to be improvised in launch week."

**How this note resolves it:** every "coming soon" string cited below lives in **hb-landing**,
not `apps/web`. Nothing in this card set edits hb-landing or changes what the public sees at
hb-ecommerce.com today. The work here is building the *new* `apps/web` pages so they never
carry "coming soon" framing to begin with — correct regardless of when launch actually
happens. The public switchover (DNS, redirects, retiring hb-landing, the announcement itself)
is a separate ops + marketing effort gated on vendor count, payment gateway, and the launch
plan from `16-marketing-strategy.md`.

**Confirmed by Michael (2026-08-13): still pre-launch.** Proceed exactly as scoped in this
note — build the pages now, launch-neutral copy, no hb-landing or public-facing change.

## Concrete before-state in hb-landing (what "coming soon" means today)

Cited so the new copy has something concrete to avoid. Do not port any of these strings.

| Location | String |
|---|---|
| `src/app/layout/header/header.html:18` | CTA button "Launching Marketplace Soon" |
| `src/app/pages/home/home.html:26` | card title "Marketplace Coming Soon" |
| `src/app/pages/home/home.html:71` | card title "Online Marketplace (Launching Soon)" |
| `src/app/pages/home/home.html:75` | disabled button "Coming Soon" |
| `src/app/pages/services/services.html:84` | whole teaser section, "Psst! Coming Soon - A Full Online Marketplace" |
| `src/app/pages/about/about.html:21` | "Starting with personal import services today, we're building toward a full trusted marketplace tomorrow." |
| `src/app/layout/footer/footer.html:12` | "...hassle-free imports and **a future marketplace**." |

Replacement framing: the marketplace is the primary offering; **on-demand sourcing/import is
a standing second service** for anything not listed on the site — explicitly cross-border
SA→Namibia.

## Scope

### Assets

Copy from hb-landing `public/` into `apps/web/public/` (currently contains only
`favicon.ico`; `angular.json` maps `public` → asset root):

- `logos/hb-logo.png`
- `images/hero-import-shopping.jpg`
- `images/about-puzzle-pieces.png`
- `images/services-shopping-cart.png`
- `images/contact-hero-image.jpg`

hb-landing indirects these through `src/app/shared/constants/image.constants.ts`
(`SITE_IMAGES` + a `cssImageUrl` helper). Mirror that pattern in `apps/web` rather than
scattering raw asset-path strings through templates. Favicons/`apple-touch-icon`/
`site.webmanifest` are optional — see Out of scope.

### Design tokens (confirmed 2026-08-13 — see Decisions)

A hybrid, not a full swap: hb-landing's raw colors and pill-button shape come across;
its Angular Material theme and font do not.

- `docs/design/DESIGN.md` + `apps/web/src/styles.scss` `--hb-primary` moves from `#015300`
  to hb-landing's raw green **`#2e7d32`**; `--hb-secondary` moves from `#964900` to
  hb-landing's raw orange **`#f57c00`** (both sourced from hb-landing's
  `--hb-green-raw`/`--hb-orange-raw`, not the Material-tone-generated variants, since no
  Material theme is being introduced).
- New button-radius token — pill-shaped primary/CTA buttons (`border-radius: 9999px`,
  matching hb-landing's global button override). `DESIGN.md`'s spacing/shape table already
  lists `full` as a defined radius value; this is that value applied to buttons specifically.
- **Do not** add `@angular/material` theming (`mat.define-theme`, `mat.$green-palette` /
  `mat.$orange-palette`) — apps/web continues to load no Material theme, per its current
  `styles.scss`.
- **Do not** change the font — stays `Inter`, not `Plus Jakarta Sans`.
- Re-check contrast: `--hb-on-primary`/`--hb-on-secondary-container` (white / dark text used
  against these fills) must still meet WCAG AA against the new hex values before shipping —
  the old and new greens/oranges are close in luminance but not identical.

### Routes (all net-new)

`/about`, `/services`, `/contact` — standalone components, signals, new control-flow syntax,
per `apps/web/CLAUDE.md`. Two mechanical constraints:

- `app.routes.ts` ends with `{ path: '**', redirectTo: 'login' }` and `''` redirects to
  `shop`. New routes go **before** the catch-all.
- `app.routes.server.ts` ends with `{ path: '**', renderMode: RenderMode.Server }`. These
  three are static marketing pages ⇒ explicit `RenderMode.Prerender` entries. **This resolves
  open question 2 in [[Public Storefront & SSR]]** ("`Prerender` reserved for future static
  marketing pages. Confirm.") — these are those pages.
- Each page sets a real `Title` + meta description. `apps/web/src/index.html` still carries
  the scaffold default `<title>HbFrontend</title>`.

### Content per page

**`/about`** — port "Our Story" and "Our Mission" from `pages/about/about.html`, rewriting
the "building toward a full trusted marketplace tomorrow" line into present tense. Ends with
the "Ready to Bridge the Gap with Us?" / "Get in Touch" CTA → `/contact`. Hero image
`about-puzzle-pieces.png`.

**`/services`** — port "Personal & Business Import Service" (h2), the three value blocks
("From Popular Sites", "For Individuals & Businesses", "Transparent & Reliable"), and the
"How It Works – Simple 4 Steps" process. **Delete the entire teaser section** at
`services.html:84` and replace it with a live cross-link into the marketplace (`/shop`) plus
a clear statement that sourcing is the complementary standing service for anything not
listed, publicly named **"Procurement Service"** (see Decisions — matches the internal name
in `07-business-model.md`, no public alias needed). Margin tiers may be stated but must match
`07-business-model.md` exactly if so. Hero image `services-shopping-cart.png`.

**`/contact`** — "Send us a Message" form + "Or Message Us on WhatsApp" + "Other Contact
Options". Contact details from hb-landing `site.constants.ts`: `+264 81 355 9921`,
`info@hb-ecommerce.com`, and the two `wa.me/264813559921` deep links. This is the "source me
something not listed on the site" channel. Submits to `POST /inquiries` (LSM-5 — see
Decisions). Hero image `contact-hero-image.jpg`.

### Footer / header wiring

`apps/web/src/app/layout/footer/footer.html` has six dead `href="#"` links across two
columns — Trade Info (Shipping Policy, Terms of Trade, Contact Support) and SME Access
(Register as Vendor, Export Documentation, Success Stories). Wire what this feature makes
real: Contact Support → `/contact`, Register as Vendor → `/vendor/apply` (already exists),
and add About Us / Services entries. Leave genuinely-unbuilt links dead rather than inventing
pages — matches the precedent set in [[Storefront UI Cleanup Batch]].

`layout/nav-bar` currently exposes one nav link ("Home" → `/shop`) plus "Sell on H&B." Add
About / Services / Contact. Note: the nav bar's `notifyComingSoon()` call is the **currency
switcher** — a legitimate unbuilt feature, unrelated to marketplace "coming soon" messaging.
Do not remove it.

## @hb/shared contract impact

**Confirmed (see Decisions — contact form wires into apps/api):**

- New `libs/shared/src/contracts/inquiry.ts` (no contact/inquiry contract exists today) —
  shape derived from hb-landing's `ContactInquiry`: `name`, `email`, `phone`, `orderType`,
  `referenceNumber`, `message`. Export from `contracts/index.ts`.
- `POST /inquiries` with a class-validator DTO **implementing** the shared interface.
- A `ContactInquiry` entity + **TypeORM migration** (`synchronize` stays off).
- Reuse `apps/api/src/mail/mail.service.ts` (Resend-backed; already has
  `sendPasswordReset`/`sendEmailVerification`) — add a public method rather than a second
  mail path.
- No money/inventory/order-state logic in this feature, so the mandatory-unit-test clause
  isn't triggered by that rule specifically — the endpoint still gets service-level tests.

## Out of scope

- **Resurrecting founder bios/photos.** `about.html`'s team section is already commented out
  and stays that way. Beyond the privacy call, the content is stale: it lists two co-founders
  who have since been bought out (`23-decisions-log.md`, closed 2026-08-04 / 2026-08-11) —
  the cap table is different now.
- **Migrating `pages/home` as a page.** `apps/web` already has `/shop` as its home; `''`
  redirects there. Only wording feeds across, not the page.
- **Migrating hb-landing's header/footer components.** `apps/web` has its own; only links and
  brand text change.
- **Favicons / `apple-touch-icon` / `site.webmanifest`** — optional, low priority, own card if
  wanted.
- **Retiring the hb-landing repo / hb-ecommerce.com deployment, DNS, redirects.** Ops concern,
  gated on the launch decision — see the Conflict section. Not this card set.
- **The `appScrollReveal` scroll-animation directive** from hb-landing — reimplement only if
  wanted, and SSR-safely (`isPlatformBrowser`). Default: don't port.
- **The launch marketing campaign** (`16-marketing-strategy.md`'s named gap) — business work,
  not code.
- Remaining dead `href="#"` footer links for genuinely unbuilt features (Shipping Policy,
  Terms of Trade, Export Documentation, Success Stories).

## Decisions (confirmed by Michael, 2026-08-13)

1. **Design tokens & typography — hybrid.** hb-landing's raw green/orange colors
   (`#2e7d32`/`#f57c00`) and pill-shaped buttons come across; its Angular Material theme and
   `Plus Jakarta Sans` font do not. `apps/web` stays theme-free (no `mat.define-theme`) and
   keeps `Inter`. Full detail in Scope → Design tokens above.
2. **"The marketplace is live" — not yet, still pre-launch.** Matches H&B Brain's roadmap
   (v1 targets 1 Oct 2026, 2/10 vendors, no gateway, no launch plan). Build all three pages
   now with launch-neutral copy that never says "coming soon" and never claims a launch
   moment; hb-landing stays untouched; the public switchover is its own, separately-gated
   effort — not part of this card set.
3. **Brand name — "H&B E-Commerce."** Sweep it across `nav-bar.html`, `footer.html` (brand
   heading + copyright line), auth-screen aria-labels/footers, vendor/admin/profile shells,
   and `environment*.ts` `appName`, replacing "H&B Market" / "H&B Marketplace" / "H&B
   Cross-Border Marketplace" everywhere they appear.
4. **Contact form — wires into `apps/api`.** New `POST /inquiries` endpoint, DB record,
   notification via the existing `MailService`. No EmailJS in the migrated app. Full detail
   in @hb/shared contract impact above (LSM-5).
5. **Public name for the sourcing engine — "Procurement Service."** Matches the internal
   name already used in `07-business-model.md`; no public-facing alias needed, no vault
   update required.

## Vertical slices → Trello cards

| # | Title | Card ID |
|---|---|---|
| LSM-1 | Migrate brand assets + settle the design-token reconciliation | kdro0zYC |
| LSM-2 | `/about` page | xhYmG5j8 |
| LSM-3 | `/services` page (sourcing as standing offering) | ahPpaEIK |
| LSM-4 | `/contact` page + WhatsApp CTA | Ah6EZCOW |
| LSM-5 | `POST /inquiries` contact endpoint (blocked on OQ4) | 9y69dIul |
| LSM-6 | Footer/nav wiring + brand-name consistency sweep | TkbgWKL2 |

Order: LSM-1 first (assets + tokens unblock everything visual). LSM-2/3/4 are independent of
each other. LSM-5 can run in parallel with LSM-4 (backend endpoint while the frontend seam is
built) or right after. LSM-6 last — it links routes that LSM-2/3/4 create and needs the
brand-name sweep to be unambiguous.


## Implementation Notes (2026-08-15)

**What shipped:** LSM-1/2/3 bundled as one branch (`feat/kdro0zYC-landing-site-migration`, five commits) because LSM-1's token swap and constants module are hard prerequisites for LSM-2/3's page logic, and LSM-2/3 both edit `app.routes.ts`/`app.routes.server.ts` (diverging branches would conflict for no isolation gain).

**Assets & tokens (LSM-1):**
- Five images copied to `apps/web/public/` (logo, hero, about, services, contact); hardcoded paths derived from hb-landing indirected through a new `SITE_IMAGES` constants module in `apps/web/src/app/shared/constants/` using absolute leading-slash paths (relative paths would resolve against the current route in an SSR app).
- Token swap completed: `--hb-primary` `#015300` → `#2e7d32`, `--hb-secondary` `#964900` → `#f57c00`, mirrored in `DESIGN.md` and `styles.scss` in lockstep. Pill-button radius (`--hb-radius-pill`, `border-radius: 9999px`) applied to primary/CTA buttons only, not as a global `button` override (hb-landing's pattern, but scoped narrower here to avoid unintended cascade). `--hb-primary-container` set to `#43a047` (hand-picked, not Material-generated; at the original `#026e00` it was darker than the new `#2e7d32` primary, inverting its documented role as lighter/hover accents).
- Token swap revealed a contrast regression: the new brighter orange and lighter green both fail AA as white-text fills. Two-round fix: (1) flipped 13 components' fills to `--hb-on-surface` text, and swapped `.checkout__submit` resting/hover states; (2) code review caught that `--hb-secondary` as a foreground colour sat at 2.70:1, then recomputed every ratio from actual hexes and found `--hb-on-secondary-container` (`#703500`, 9.55:1) was the documented fit. Documented `--hb-secondary` as fills/bars only to prevent recurrence.
- **Key lesson:** colour-token swaps need contrast re-audited on both fills and foregrounds, separately, as two distinct passes.

**Pages (LSM-2 & LSM-3):**
- `/about` prerendered: ported "Our Story" / "Our Mission", rewrote "building toward a full trusted marketplace tomorrow" to present tense (launch-neutral copy per spec requirement).
- `/services` prerendered: ported the three value blocks and 4-step process, **deleted the entire teaser section** and replaced it with a live `/shop` cross-link stating the dual-engine model explicitly (marketplace is primary, sourcing is a standing complementary service for unlisted items). Margin tiers are **not** published on `/services` (transparency promised, pricing withheld per spec rule 4).
- `TrustBanner` component gained an optional `label` input so its `aria-label` can describe what it labels per page (reused across `/services` value blocks rather than adding a second 3-card grid).
- Both pages carry explicit `RenderMode.Prerender` entries in `app.routes.server.ts` — this resolves [[Public Storefront & SSR]] open question 2 (confirmed these as the static marketing pages `Prerender` was reserved for).

**Deferred out of scope:**
- `/contact` (LSM-4) not built; both pages' `routerLink="/contact"` falls through catch-all to `/login` until it lands.
- Nav/footer wiring and brand-name sweep to "H&B E-Commerce" are LSM-6; nav and footer still read "H&B Market".
- **Asset weight unaddressed:** the copied images total ~6.4 MB (hb-logo.png alone 902 KB, 2000×2000 PNG rendered at 32×32 on every route). LSM-1's AC required only copying them; a follow-up card owns optimization (resize/re-encode, `NgOptimizedImage` evaluation). This undercuts the point of prerendering for a mobile-data audience.

**Verification:** 837/837 Vitest specs pass; `npm run build` clean, prerendering both routes; both pages verified live against dev server with no console errors; all links wired except `/contact`.
