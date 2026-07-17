---
operator: Josh
date: 2026-07-17
session: 10
tags: [ai-factory, ship-batch, analytics]
---

# Session 10 — Analytics & Reporting batch

## What we worked on

Trello cards **LKIGKUVM, FRtcp2dg, p0kVZMii, UC508IB6, ezkaNtbZ** — Analytics & Reporting slices 1–5, bundled in a single `/ship-batch` PR. Shipped first-party analytics event ingestion, storefront instrumentation, GA4 consent-gating, admin funnel/revenue dashboards, and vendor-scoped analytics.

## What changed

**Backend (`apps/api/src/analytics` + `libs/shared/src/contracts/analytics.ts` + `libs/shared/src/enums/analytics-event-type.ts`):**
- New `@hb/shared` contracts: CreateAnalyticsEventRequest (sessionId, optional productId/vendorId/orderId, value+currency required-together), AnalyticsEventDto, AdminAnalyticsSummaryDto (funnel counts, conversion rate, revenue-by-currency time series), VendorAnalyticsDto (vendor-scoped funnel, orders, gross-GMV). AnalyticsEventType enum (7 stages: PRODUCT_VIEWED, ADD_TO_CART, CHECKOUT_STARTED, SHIPPING_SUBMITTED, PAYMENT_ATTEMPTED, PAYMENT_FAILED, ORDER_COMPLETED).
- `@Public POST /analytics/events` ingestion endpoint — class-validator DTO enforces sessionId required, value+currency required-together, metadata capped 2 KiB. Rejects spoofed currency mismatches.
- New `analytics_events` TypeORM entity + migration 1784246400000. Columns: id/type/sessionId/userId/productId/vendorId/orderId/value/currency/metadata/occurredAt/receivedAt. Indexes on type/sessionId/vendorId. Numeric(12,2) + explicit currency_code. No PII. No FKs (denormalized by design, matches AD-001 projection target).
- `GET /admin/analytics` (@Roles ADMIN) — aggregates distinct-session funnel stage counts (zero-filled for missing stages), zero-safe conversion rate, orders count + line-GMV revenue-by-currency over time (day/ISO-week/month buckets). Follows AP-10 money rules; unit-tested bucketing helpers (incl. Sunday ISO-week edge case, quantity>1 GMV, float-drift rounding, value:0 boundary).
- `GET /vendors/me/analytics` (@Roles VENDOR) — vendor-scoped (service layer resolves userId → vendorId, no hand-rolled filters). Funnel counts, ADD_TO_CART count, orders count, gross-GMV revenue-by-currency, time series. Strict tenant isolation; unit-tested.
- Shared `common/utils/analytics-bucket.ts` range/bucketing helpers extracted; used by both admin + vendor endpoints.
- Public-routes guardrail updated; analytics endpoint blessed.

**Frontend (`apps/web/src/app/common/services/analytics/` + dashboard widgets):**
- New SSR-safe `AnalyticsService` (isPlatformBrowser guard, fire-and-forget, persistent anonymous localStorage sessionId UUID, in-memory fallback for private-mode browsers). Generates sessionId once on first call; survives page reloads.
- Wired all 7 funnel events: PRODUCT_VIEWED (product-detail), ADD_TO_CART (cart update), CHECKOUT_STARTED (checkout route), SHIPPING_SUBMITTED (shipping form submit), PAYMENT_ATTEMPTED (payment init), PAYMENT_FAILED (payment error), ORDER_COMPLETED (order confirmation).
- New `ConsentService` — minimal POPIA consent banner (gates GA only). No granular cookie categories. First-party event ingestion carries no consent gate (operational data).
- New `GoogleAnalyticsService` — inert until `environment.gaMeasurementId` is provisioned (currently ''). Maps core GA4 e-commerce events (view_item, add_to_cart, begin_checkout, purchase) to funnel stages. No PII. SSR-safe.
- Admin dashboard analytics section: distinct-session funnel bar chart, conversion-rate gauge, revenue-by-currency stacked bar (ZAR/NAD separate), orders time-series sparkline. Pure CSS bars; DESIGN.md tokens.
- Vendor dashboard analytics section: funnel counts card, gross-GMV revenue-by-currency, time series. Vendor-scoped, reads from `GET /vendors/me/analytics`.

## Key decisions

1. **Direct ingestion endpoint built now** — ADR-001 backbone is Proposed, not built. Unblocks analytics feature immediately. Event type names align with future domain events (PRODUCT_VIEWED, ORDER_COMPLETED, etc.) so no rework when backbone projects analytics later.
2. **Session ID = analytics-only persistent UUID** — no collision with auth/cart ids. Simplest source-of-truth for funnel attribution.
3. **GA4 gated by consent, first-party events are not** — POPIA requires consent for GA4; first-party events are operational (order/checkout funnel is part of the purchase flow, not behavioural tracking). Minimal banner confirms visitor's GA choice.
4. **Admin KPIs v1 = funnel + conversion + revenue-over-time** — proposed list confirmed; no additional KPIs requested.
5. **Vendor revenue = gross GMV** — net-of-commission deferred; commission structure in BRS §9.5 still TBD. Vendors will see gross only for now.
6. **No GA4 property exists yet** — shipped inert. Measurement ID goes in `environment.ts` when provisioned; no blocking dependency.
7. **Trello MCP was down this session** — REST fallback used (creds from .mcp.json); card comments + moves handled via API directly.

## Scope & orchestration

- Five cards (LKIGKUVM, FRtcp2dg, p0kVZMii, UC508IB6, ezkaNtbZ) bundled via `/ship-batch` into one branch/PR.
- Sequential backend slices 1→2→3 (contracts+ingestion → web instrumentation → GA4+consent), parallel 4api/5api (admin/vendor analytics disjoint), then dashboard widgets. Three agents (backend-engineer, frontend-engineer specialising in services, frontend-engineer on dashboard), one test-engineer, one code-reviewer.
- 10 commits total: shared contracts, POST /analytics/events, analytics_events entity, web AnalyticsService, funnel event wiring, ConsentService, GoogleAnalyticsService, GET /admin/analytics, GET /vendors/me/analytics, admin/vendor dashboard widgets.

## Test/review outcome

- **API:** 397/397 tests pass (100%, +40 new test cases: analytics-bucket utils edge cases, admin funnel aggregation, vendor tenant isolation, GA4 event mapping, consent state).
- **Web:** 545/545 tests pass (100%, +32 new: AnalyticsService sessionId persistence, funnel event firing, SSR safety checks, ConsentService state).
- **Lint:** clean.
- **Build:** full SSR build clean.
- **Test-engineer audit:** no coverage gaps; all money/inventory/order logic tested. Four real gaps closed: bucketing utils direct coverage incl. Sunday ISO-week edge case, quantity>1 line GMV, float-drift rounding, value:0 boundary.
- **Code-reviewer:** SHIP, 0 FAIL / 2 WARN (both fixed in pre-PR commit):
  - Token-derived focus rings on consent-banner dismiss button — switched to `--hb-primary` token.
  - Metadata size cap validation — clarified max 2 KiB enforced; error message tightened.

## PR status

PR #34 — https://github.com/michaeljvr11/hb-mono-repo/pull/34 — open, awaiting human merge (lead owns prod).

## Evidence snapshot

After `npm run evidence` (2026-07-17):
- **139 commits** (2026-06-11 → 2026-07-17)
- **129** carry AI-authorship trailer
- **80** specs compiled
- **17** fence blocks (guardrails)

## Follow-ups

- Provision GA4 property; set `gaMeasurementId` in `apps/web/src/environments/environment.ts`.
- Deferred polish (shared AnalyticsRangeQuery interface, funnel-bar helper component extraction, GaItem type merge, ~45 lines of dedup).
- DB-side GROUP BY for order bucketing if admin analytics query latency ever surfaces.
- Consent UX hardening if marketing needs granular cookie categories.

## Deliverables (this session)

- **Analytics & Reporting.md** Implementation Notes section — what shipped per slice, resolved open questions, orchestration, follow-ups.
- **History/Josh/session-10.md** (this file) — batch summary, test/review outcome, PR link, evidence snapshot.
- **AI Factory — Evidence Log.md** — refreshed headline figures (139 commits · 129 AI-tagged · 80 specs · 17 fence blocks) + PR #34 entry.
