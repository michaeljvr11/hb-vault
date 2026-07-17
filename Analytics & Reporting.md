# Analytics & Reporting

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Order State Machine]] · [[Money & Currency Rules]] ·
[[Vendor & Admin Portals]] · [[Public Storefront & SSR]] · [[Listing Types & Vendor Rules]].

Companion git-tracked docs: `docs/business/BRS.md` (FR-ADM-3, NFR-6),
`docs/architecture/ADR-001-event-driven-backbone.md`.

## Problem

HB has no product-analytics layer. Admins get static read-model counts
(`AdminDashboardDto`) and vendors get a single-KPI snapshot (`VendorDashboardDto`),
but nobody can see **how customers move through the funnel**, where they drop off, or
how conversion/revenue trend over time. The product owner wants two complementary
pillars:

1. **Google Analytics (GA4)** on the Angular storefront — page-view + standard
   e-commerce behavioural events (traffic, browse behaviour, funnel drop-off).
2. **Bespoke first-party analytics** we build ourselves, hooked into the checkout
   process — funnel + conversion + revenue events persisted in our own Postgres, so
   the numbers are ours (survive GA sampling/consent gaps, and are join-able to real
   orders/vendors for money-accurate reporting).

These feed two surfaces that **already exist** (do NOT rebuild them):
- **Admin Portal dashboards** — platform-wide funnel/revenue (`/admin/dashboard`).
- **Vendor Portal analytics** — each vendor sees only their own product performance
  (`/vendor/dashboard`), under the same service-layer ownership rule as the rest of
  the vendor portal.

## Goals

- Capture the storefront → checkout → order funnel as first-party events we own.
- Surface a funnel + conversion + revenue-over-time view to admins (platform-wide).
- Surface a vendor-scoped analytics view (own products only) to vendors.
- Add GA4 for behavioural/traffic analytics, SSR-safe and consent-gated.
- Keep money reporting currency-explicit (ZAR and NAD never summed — see
  [[Money & Currency Rules]]).

## Scope

1. **`@hb/shared` analytics contracts** — event-type enum, event ingest request DTO,
   stored-event DTO, admin analytics-summary DTO, vendor analytics DTO. Interfaces +
   enums only; API DTOs `implement` them; web consumes them.
2. **Bespoke event ingestion** — `POST /analytics/events` (`@Public`, storefront is
   anonymous until checkout) with a class-validator DTO; `analytics_events` entity +
   TypeORM migration; write path unit-tested.
3. **Storefront checkout instrumentation** — the storefront fires bespoke funnel
   events (view → add-to-cart → begin-checkout → payment attempt → order completed)
   through the ingestion endpoint. SSR-safe (browser-only execution).
4. **GA4 storefront integration** — SSR-safe, consent-gated GA service emitting the
   core GA4 e-commerce events (`view_item`, `add_to_cart`, `begin_checkout`,
   `purchase`). Measurement ID from env config, never hard-coded.
5. **Admin analytics** — aggregation endpoint(s) + dashboard widgets: funnel counts +
   conversion rate, revenue-by-currency over time, orders-over-time. Money aggregation
   is unit-tested.
6. **Vendor analytics** — vendor-scoped aggregation endpoint + vendor dashboard view;
   strict tenant scoping (vendor sees only own products/lines), service-layer ownership,
   unit-tested.

## Business rules it must honour

From [[Money & Currency Rules]], [[Vendor & Admin Portals]], [[Order State Machine]],
[[Public Storefront & SSR]] and `docs/business/BRS.md` — enforce, do not reinvent:

- **Money stays `numeric(12,2)` + explicit currency; ZAR/NAD are reported separately,
  never summed.** Any revenue metric carries its `CurrencyCode`. Peg is data. Any
  service method that sums money gets a unit test (NFR-1 / [[Money & Currency Rules]]).
  Mirror the existing `CurrencyTotalDto` pattern in `contracts/dashboard.ts`.
- **Order revenue = line-item GMV snapshot** (`unitPrice * quantity` recorded at order
  time), grouped by currency, cancelled orders excluded — matching the shipped
  `AdminDashboardDto` / `VendorDashboardDto` behaviour. Don't recompute from live prices.
- **Vendor-scoped queries MUST filter by ownership in the service layer** — a vendor
  sees only analytics for their own `vendorId` lines/products. Never hand-roll the
  filter in a controller ([[Vendor & Admin Portals]]).
- **Funnel semantics follow the order lifecycle** in [[Order State Machine]]
  (`pending → confirmed → processing → shipped → delivered | cancelled`). Bespoke
  event types name funnel stages, not order statuses — keep them decoupled.
- **SSR safety is non-negotiable** ([[Public Storefront & SSR]]): all GA and event-firing
  code is `isPlatformBrowser`-guarded / runs only in `afterNextRender`; no `window`/
  `document`/`gtag` access on the server. The storefront is `RenderMode.Server`.
- **POPIA (ZA) applies to personal data** (`docs/business/BRS.md` NFR-6): GA is
  consent-gated; **no PII in GA or bespoke events** — key bespoke events to
  session/anonymous ids and (post-auth) `orderId`/`userId` references we already store,
  not names/emails. Audit-log privacy discipline applies here too.
- **Contract integrity** (NFR-2): every endpoint input is a class-validator DTO
  implementing the `@hb/shared` interface; no duplicated types.

## @hb/shared contract impact

New `contracts/analytics.ts` (interfaces only) + enum. Reuse `CurrencyCode`,
`CurrencyTotalDto`, `OrderStatus`, `ListingType`, `CountryCode` — do not duplicate.

- **`enums/analytics-event-type.ts`** (new, const-object pattern) — bespoke funnel
  event types, e.g. `PRODUCT_VIEWED`, `ADD_TO_CART`, `CHECKOUT_STARTED`,
  `SHIPPING_SUBMITTED`, `PAYMENT_ATTEMPTED`, `PAYMENT_FAILED`, `ORDER_COMPLETED`.
  (Confirm final list — see open questions.)
- **`CreateAnalyticsEventRequest`** — ingest payload: `type`, `sessionId` (anonymous),
  optional `productId` / `vendorId` / `orderId`, optional `value` + `currency`
  (`CurrencyCode`, required whenever `value` is present), optional `metadata`
  (`Record<string, unknown>`, no PII), client `occurredAt`.
- **`AnalyticsEventDto`** — stored shape (adds `id`, server `receivedAt`).
- **`AdminAnalyticsSummaryDto`** — funnel stage counts, conversion rate(s),
  `revenueByCurrency: CurrencyTotalDto[]`, and a `TimeSeriesPointDto[]`
  (date bucket → orders count + revenue-by-currency). Query DTO for date range +
  granularity.
- **`VendorAnalyticsDto`** — vendor-scoped: own funnel/`ADD_TO_CART`/orders counts,
  `revenueByCurrency: CurrencyTotalDto[]`, time series — all restricted to the
  authenticated vendor's lines.

Every new endpoint input = a class-validator DTO implementing the shared interface.

## Data & privacy

- **Bespoke events** land in a new `analytics_events` table (TypeORM migration;
  `synchronize` stays off). Suggested columns: `id` uuid PK, `type` varchar (indexed),
  `sessionId` varchar (indexed), `userId` uuid null, `productId` uuid null,
  `vendorId` uuid null (indexed), `orderId` uuid null, `value numeric(12,2)` null,
  `currency` currency_code null, `metadata` jsonb null, `occurredAt` timestamptz,
  `receivedAt` timestamptz. `value` uses `numeric(12,2)` + explicit `currency` per
  [[Money & Currency Rules]] — but **authoritative revenue is aggregated from
  `orders`/`order_items`, not from event `value`** (events can be spoofed / lost);
  event `value` is for funnel-value/GA parity only.
- **No PII** in events or GA. Bespoke events keyed to an anonymous `sessionId`
  pre-auth, and to `orderId`/`userId` (ids we already hold) post-auth. GA events carry
  no email/name/address.
- **Consent (POPIA):** GA only initialises after the visitor accepts analytics cookies.
  Bespoke first-party events are operational/first-party but the consent posture for
  them is an open question below.
- **Currency:** every revenue figure carries `CurrencyCode`; ZAR and NAD are separate
  series; the 1:1 peg is never assumed in aggregation.

## Out of scope (recorded so nobody assumes them)

- Data warehouse / BI export / CSV download / scheduled email reports.
- Real-time streaming dashboards / websockets (aggregation is request-time query).
- Server-side GA Measurement Protocol (client-side GA4 only for now).
- A cookie-consent banner **build** — see open questions; if none exists, GA is gated
  behind a consent flag but the banner UI may be a separate card / human decision.
- Rebuilding the admin or vendor dashboards — these exist; analytics widgets slot in.
- Re-platforming analytics onto the ADR-001 event backbone — see open questions; the
  ingestion endpoint is the pragmatic first step and can later become an event
  projection consumer without changing the contracts.
- Commission/settlement-aware "net vendor revenue" — commission structure is TBD
  (`docs/business/BRS.md` §9.5, [[Money & Currency Rules]] TBD). Vendor revenue is
  **gross GMV** until a human confirms otherwise.

## Open questions (confirm before /ship-card)

1. **GA4 property ownership + Measurement ID** — who owns the GA4 property; what env
   var holds the measurement id (goes in web `environment`, non-secret, like
   `apiBaseUrl`)? Blocks card 3.
2. **Consent banner** — does a POPIA cookie-consent mechanism exist anywhere in
   `apps/web`? (Grep found none.) If not, is a minimal consent gate in scope for
   card 3, or a prerequisite card / human decision? GA must not fire without consent.
3. **Which KPIs do admins want first?** Proposed v1: funnel counts + overall
   conversion rate, orders/day, revenue/day by currency. Confirm the priority list.
4. **Vendor revenue definition** — **gross GMV** (assumed) vs **net of commission**?
   Commission structure is unresolved (BRS §9.5). Confirm before card 5 ships numbers
   vendors will treat as settlement.
5. **Bespoke analytics vs ADR-001 event backbone** — ADR-001 (Proposed, not built)
   introduces a domain-event outbox + `hb.orders`/`hb.payments` topics and calls out
   an "audit/analytics surface" as a projection (FR-ADM-3). Should bespoke analytics be
   built now as a **direct ingestion endpoint** (recommended: unblocks the feature, and
   the funnel/browse events are front-end behavioural, not domain events), and later
   *augmented* by a server-side order/payment projection when the backbone lands? Name
   bespoke event types to align with future domain events to minimise rework.
6. **Anonymous session id source** — reuse an existing client id/cookie, or mint an
   analytics-only anonymous id? (Affects card 1 DTO + card 2.)

## Repo reality check (important for /ship-card)

- **The current working branch (`fable-storefront-oneshot`) is behind `main`.** On this
  branch `apps/api/src/orders/orders.service.ts` is still a skeleton (only
  `findAllForUser`), the web `cart`/`checkout` routes are commented out in
  `app.routes.ts`, and `OrderStatus` lacks `handed_to_hb`. But `docs/business/BRS.md §4`
  and `docs/business/ROADMAP.md` (Phase 0) state — "verified against code on `main`" —
  that server-side cart, a checkout page, order creation, and the state machine are
  **already merged on `main`**. `/ship-card` MUST diff `origin/main` first to find the
  real checkout components/service to instrument (cards 2 & 4 depend on them). Do not
  anchor to this branch's skeleton.
- Existing read-models to extend, not duplicate: `AdminDashboardDto` +
  `CurrencyTotalDto` (`libs/shared/src/contracts/dashboard.ts`), `VendorDashboardDto`
  (`libs/shared/src/contracts/vendor.ts`), `GET /admin/dashboard`,
  `GET /vendors/me/dashboard`.
- No analytics business rules exist in the vault yet — this note is the first. Nearest
  prior art: `AdminDashboardDto` revenue-by-currency aggregation (card AP-10) and
  `VendorDashboardDto` (card VP-2), both of which set the currency + ownership patterns
  the analytics aggregation must follow.

## Vertical slices → Trello cards

1. `@hb/shared` analytics contracts + bespoke event ingestion + `analytics_events`
   entity/migration (API slice, tests).
2. Storefront checkout instrumentation — fire bespoke funnel events (web → API).
   Depends on 1.
3. GA4 storefront integration — SSR-safe, consent-gated, core e-commerce events.
   Depends on open questions 1 & 2.
4. Admin analytics — aggregation endpoint(s) + dashboard widgets (funnel, conversion,
   revenue-by-currency over time). Depends on 1 (and real checkout on `main`).
5. Vendor analytics — vendor-scoped aggregation endpoint + vendor dashboard view
   (strict tenant scoping, tests). Depends on 1.


## Implementation Notes (2026-07-17 — /ship-batch, PR #34)

**Cards shipped:** LKIGKUVM, FRtcp2dg, p0kVZMii, UC508IB6, ezkaNtbZ bundled in branch `feat/LKIGKUVM-analytics-reporting` → PR #34 (https://github.com/michaeljvr11/hb-mono-repo/pull/34). 10 commits, awaiting human merge.

**What shipped per slice:**

1. **Card LKIGKUVM** — `@hb/shared` analytics contracts. New `enums/analytics-event-type.ts` (7 stages: PRODUCT_VIEWED, ADD_TO_CART, CHECKOUT_STARTED, SHIPPING_SUBMITTED, PAYMENT_ATTEMPTED, PAYMENT_FAILED, ORDER_COMPLETED); new `contracts/analytics.ts` (CreateAnalyticsEventRequest, AnalyticsEventDto, AdminAnalyticsSummaryDto, VendorAnalyticsDto — all reuse CurrencyCode, CurrencyTotalDto, OrderStatus). API ingestion endpoint `@Public POST /analytics/events` with class-validator DTO (sessionId required, productId/vendorId/orderId optional, value+currency required-together, metadata capped 2 KiB). New `analytics_events` entity (TypeORM) + migration 1784246400000 (columns: id/type/sessionId/userId/productId/vendorId/orderId/value/currency/metadata/occurredAt/receivedAt, indexes on type/sessionId/vendorId, numeric(12,2) + explicit currency_code, no PII, no FKs by design). Public-routes guardrail updated.

2. **Card FRtcp2dg** — Web instrumentation. SSR-safe AnalyticsService (isPlatformBrowser guard, fire-and-forget, persistent anonymous localStorage sessionId, in-memory fallback for private-mode). All 7 funnel events wired at product-detail/discover/shop/checkout routes. No server-side GA Measurement Protocol.

3. **Card p0kVZMii** — GA4 integration + consent. New ConsentService gates GA only (minimal POPIA consent banner, no granular categories). GoogleAnalyticsService inert until `environment.gaMeasurementId` is provisioned (currently ''). Core GA4 events (view_item, add_to_cart, begin_checkout, purchase) mapped to funnel stages, no PII, SSR-safe.

4. **Card UC508IB6** — Admin analytics. GET /admin/analytics (@Roles ADMIN) aggregates distinct-session funnel counts (zero-filled), zero-safe conversion rate, orders+line-GMV time series (day/ISO-week/month buckets). Follows AP-10 money rules (numeric(12,2) + explicit currency, ZAR/NAD separate). Admin dashboard analytics widgets (pure CSS bars, DESIGN.md tokens).

5. **Card ezkaNtbZ** — Vendor analytics. GET /vendors/me/analytics (@Roles VENDOR) vendor-scoped to authenticated userId (service-layer isolation, VP-2 pattern). Funnel counts, ADD_TO_CART count, orders count, gross-GMV revenue by currency, time series. Shared range/bucketing helpers extracted to `common/utils/analytics-bucket.ts`.

**Resolved open questions** (mapping to the note's "Open questions" section):

1. **GA4 Measurement ID** — none exists; shipped inert. Measurement ID goes in `apps/web/src/environments/environment.ts` when GA4 property is provisioned. No blocking dependency.
2. **Consent banner** — minimal banner built, gates GA only; no granular cookie categories. POPIA-compliant first-party event ingestion carries no banner gate (operational data). Future consent UX polish out of scope.
3. **Admin KPIs v1** — proposed funnel + conversion + revenue-over-time confirmed. No additional KPIs requested.
4. **Vendor revenue** — gross-GMV confirmed (net-of-commission deferred; commission structure in BRS §9.5 still TBD).
5. **Bespoke analytics vs ADR-001 event backbone** — direct ingestion endpoint built now (unblocks feature). Event names align with future backbone projections (PRODUCT_VIEWED, ORDER_COMPLETED, etc.); no rework needed if ADR-001 domain events later become analytics projection consumers.
6. **Anonymous session ID source** — analytics-only persistent UUID minted in web AnalyticsService; no collision with existing auth/cart ids.

**Orchestration:** Sequential slices 1→2→3→4api→5api, then admin/vendor dashboard widgets in parallel (disjoint files). Two agents (backend-engineer, frontend-engineer), one test-engineer (4 real gaps closed: bucketing utils direct coverage incl. Sunday ISO-week edge case, quantity>1 line GMV, float-drift rounding, value:0 boundary), one code-reviewer. SHIP 0 FAIL / 2 WARN (both fixed pre-PR: token-derived focus rings in consent banner, metadata size cap validation tightened).

**Test outcome:** API 397/397 (31 suites), web 545/545 (49 files), lint clean, full SSR build clean.

**Follow-ups:** Provision GA4 property + set gaMeasurementId in `environment.ts`; deferred polish (shared AnalyticsRangeQuery interface, funnel-bar helper component extraction, GaItem type merge, ~45 lines of dedup); DB-side GROUP BY for order bucketing if admin analytics query latency surfaces; consent UX hardening if marketing needs granular categories.
