# Landing Site Migration

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Status: **specced 2026-08-13**, awaiting decisions OQ1–OQ5 before `/ship-card`.
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
plan from `16-marketing-strategy.md`. See OQ2 below — **confirm with Michael before treating
"the marketplace is live" as fact anywhere public-facing.**

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
listed. Margin tiers may be stated but must match `07-business-model.md` exactly if so. Hero
image `services-shopping-cart.png`.

**`/contact`** — "Send us a Message" form + "Or Message Us on WhatsApp" + "Other Contact
Options". Contact details from hb-landing `site.constants.ts`: `+264 81 355 9921`,
`info@hb-ecommerce.com`, and the two `wa.me/264813559921` deep links. This is the "source me
something not listed on the site" channel. Submit target is **OQ4**. Hero image
`contact-hero-image.jpg`.

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

**None — unless OQ4 resolves to "backend."** If it does:

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

## Open questions (confirm before `/ship-card`)

1. **Design tokens & typography — which system wins?** hb-landing is `mat.$green-palette` /
   `mat.$orange-palette`, `Plus Jakarta Sans`, and pill buttons (`border-radius: 9999px` on
   every `mat-button` variant). `docs/design/DESIGN.md` (canonical, Claude Design, migrated
   2026-06-18) is `--hb-primary #015300`, `--hb-secondary #964900`, `Inter`, radius 8/12px.
   **Recommendation: Claude Design tokens win** — logo, imagery, and copy migrate as-is and
   get reskinned to match. Not just because it's the documented source: `apps/web` currently
   loads **no Angular Material theme at all** (`styles.scss` says so explicitly — it's what
   caused the snackbar token bug fixed in commit `96a2a3d`). hb-landing's look is delivered
   *through* a Material theme, so adopting it means introducing one and restyling every
   Material control across storefront, checkout, vendor, and admin portals (7 implemented
   screens), diverging from the `docs/design/claude-design/` sync bundle in the process.
   Softening note: `--hb-secondary #964900` / `--hb-secondary-fixed #ffdcc7` already *is* an
   orange earth-tone, so the green+orange pairing largely survives either way. If the
   pill-button signature specifically is wanted, that's a small additive radius-token change
   independent of the theme question. **Needs Michael's confirmation before LSM-1 starts.**
2. **Do we actually say "the marketplace is live" anywhere public-facing?** See the Conflict
   section — v1 targets 1 Oct 2026, 2/10 vendors, no gateway, no launch plan.
   **Recommendation:** build all three pages now with launch-neutral copy that never says
   "coming soon" and never claims a launch moment; leave hb-landing untouched; treat the
   public switchover as its own, separately-gated effort.
3. **Brand name — apps/web currently ships four variants.** "H&B Market" (nav-bar, footer,
   vendor/admin/profile shells), "H&B Cross-Border Marketplace" (footer copyright, login
   footer), "H&B Marketplace" (auth aria-labels), "H&B E-Commerce" (`environment.ts`
   `appName`, and the live domain). No canonical answer exists in H&B Brain.
   **Recommendation:** pick one customer-facing wordmark and sweep it everywhere. Pure
   founder call.
4. **Contact form: keep EmailJS, or wire it into `apps/api`?** hb-landing's
   `contact.service.ts` posts client-side to EmailJS with `serviceId`/`templateId`/
   `publicKey` hardcoded, no backend call at all. **Recommendation: wire it to `apps/api`.**
   Porting EmailJS as-is would violate `apps/web/CLAUDE.md` ("Frontend env files hold only
   `apiBaseUrl` + flags. No secrets, no provider keys"); `MailService` already exists to
   reuse; and a persisted inquiry gives `15-customer-support.md`'s "simple shared issue log"
   (an accepted gap today) nearly for free. Cost: a new shared contract, DTO, entity,
   migration, endpoint (LSM-5). Cheaper interim: keep EmailJS behind a `ContactService` seam
   and swap the transport later without touching the component.
5. **Public name for the sourcing engine.** H&B Brain calls it "Procurement Service";
   hb-landing calls it "Personal & Business Import Service." **Recommendation:** pick one
   public-facing name for `/services` and the footer; if it differs from the internal name,
   update `07-business-model.md` to note the public-facing alias.

## Vertical slices → Trello cards

| # | Title | Card ID |
|---|---|---|
| LSM-1 | Migrate brand assets + settle the design-token reconciliation | kdro0zYC |
| LSM-2 | `/about` page | xhYmG5j8 |
| LSM-3 | `/services` page (sourcing as standing offering) | ahPpaEIK |
| LSM-4 | `/contact` page + WhatsApp CTA | Ah6EZCOW |
| LSM-5 | `POST /inquiries` contact endpoint (blocked on OQ4) | 9y69dIul |
| LSM-6 | Footer/nav wiring + brand-name consistency sweep | TkbgWKL2 |

Order: LSM-1 first (assets + token decision unblock everything visual). LSM-2/3/4 are
independent of each other. LSM-5 only if OQ4 resolves to "backend." LSM-6 last — it links
routes that LSM-2/3/4 create.
