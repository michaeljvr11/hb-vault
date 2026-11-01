# Business Requirements Specification (BRS)

**Project:** H&B Cross-Border Commerce Platform (South Africa → Namibia)
**Document status:** Draft v0.1 — for review
**Date:** 2026-07-07
**Author:** AI-drafted (Claude), pending human review
**Audience:** Tech Accelerator judges, HB product owners, engineering team

> Canonical business rules live in the Obsidian vault. This BRS is the git-tracked,
> judge-visible consolidation; once approved, its requirement IDs feed `/spec-feature`
> to produce the Obsidian spec notes and Trello cards that drive implementation.

---

## 1. Executive summary

H&B ("HB") bridges retail supply between South Africa (ZA) and Namibia (NA). Today the
business runs a personal/business import service (manual quotes, 24–48h response). The
platform under construction digitizes that service and extends it into two coexisting
business models on one codebase:

1. **Platform-fulfilled commerce** — HB sources, lists, and delivers products itself.
2. **Vendor marketplace** — verified ZA SMEs list and fulfil their own products.

ZA and NA share the SACU customs union and a 1:1-pegged currency (ZAR/NAD). The system
treats the peg as **data, never an assumption**, and treats the border crossing
(`at_border`, `customs_cleared`) as a first-class business step.

This BRS also motivates the platform's next architectural step: an **event-driven
backbone** (see [[ADR-001 Event-Driven Backbone|ADR-001]]) to power
order tracking, notifications, search indexing, and the audit/analytics surface that a
two-sided cross-border marketplace requires.

## 2. Business objectives

| ID | Objective | Success measure |
|----|-----------|-----------------|
| BO-1 | Launch a self-service storefront for NA customers buying ZA goods | Customer can browse → cart → checkout → pay → track without human intervention |
| BO-2 | Onboard verified ZA vendors into a marketplace | Vendor can register, be approved, list products, and see their orders |
| BO-3 | Make the cross-border leg transparent | Customer sees border/customs milestones on every shipment |
| BO-4 | Digitize the existing import-request/quote service | Quote requests captured in-platform, response SLA ≤ 48h tracked |
| BO-5 | Give HB ops a single operational view | Admin portal covers catalog, vendors, orders, payments, shipments, audit |
| BO-6 | (Hackathon) Demonstrate an auditable AI software factory | Evidence pack compiles; every AI change traced card → branch → PR |

## 3. Stakeholders

| Stakeholder | Interest |
|---|---|
| NA customers (personal & business buyers) | Reliable sourcing, transparent landed cost, delivery visibility |
| ZA vendors (verified SMEs) | Sales channel into NA, simple onboarding, timely settlement |
| HB operations / admin | Catalog control, vendor vetting, order & customs oversight |
| Logistics partners (future courier integration) | Clean shipment handoffs, status callbacks |
| Payment providers (future: Payfast/Paystack class) | Standard capture/refund integration behind the existing port |
| Customs / SACU compliance | Correct documentation per shipment (`customsReference`) |
| Tech Accelerator judges | Process evidence: planning, requirements, testing, review, docs |

## 4. Current state (as-is, verified against code on `main`)

Implemented and merged:

- **Auth & users** — JWT access + rotating hashed refresh token (httpOnly cookie), global guards, roles (`admin`, vendor, customer).
- **Catalog** — products (platform vs vendor listings enforced by DB CHECK), categories, images, currency + origin country per product; unified search module; discovery + product detail pages (SSR Angular storefront).
- **Cart & checkout** — server-side cart with live pricing and ownership checks; checkout page with verification/stock error affordances.
- **Orders** — order creation with state machine; statuses `pending → confirmed → processing → shipped → delivered | cancelled`; order items snapshot `listingType` + `vendorId`; cross-border pair (`originCountry`, `destinationCountry`) on every order.
- **Payments** — `PAYMENT_PROVIDER` port with stub capture; statuses `pending / authorized / paid / failed / refunded`. **No real provider yet (deliberate).**
- **Shipping** — `SHIPPING_PROVIDER` port + stub; `ShipmentStatus` includes `at_border` and `customs_cleared`. **No real courier yet (deliberate).**
- **Admin & audit** — admin module, audit log module, mail module.
- **Platform** — npm-workspaces monorepo, `@hb/shared` contract lib, TypeORM migrations (`synchronize` off), Postgres 16, security-hardening pass (2026-07-02 audit).

Explicitly **not** implemented (ports ready): real payment capture, real courier booking, KYC, FX conversion, customs document generation, vendor settlement/payouts.

## 5. Scope

### 5.1 In scope (this planning horizon)
- Completing the transactional core: payment adapter (sandbox), fulfilment states, vendor order views.
- Event-driven backbone per ADR-001: domain events, transactional outbox, broker, consumers (notifications, search indexing, audit projection, shipment timeline).
- Customer-facing order tracking incl. border milestones.
- Import-request / quote capture flow (digitizing the current live service).
- Admin operational dashboards fed by events.

### 5.2 Out of scope (recorded so nobody assumes them)
- FX conversion logic (peg is 1:1 data; revisit only if the peg changes).
- Full vendor KYC/verification workflow beyond status management.
- Microservice decomposition — the API remains a **modular monolith**; events are the seam, not an excuse to split services.
- Multi-warehouse inventory, returns/RMA, loyalty.

## 6. Functional requirements

Priorities: **M**ust / **S**hould / **C**ould (MoSCoW). "Event" column marks requirements that consume or emit domain events (the concrete justification for ADR-001).

### 6.1 Catalog & search
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-CAT-1 | Vendors manage only their own listings; admins manage platform listings (existing constraint: vendor listings must have a vendor, platform listings must not) | M | — |
| FR-CAT-2 | Product create/update/archive re-indexes search automatically | M | ✔ `catalog.product.updated` |
| FR-CAT-3 | Customers filter/search across both listing types with country/currency shown explicitly | M | — |

### 6.2 Cart, checkout & orders
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-ORD-1 | Checkout re-verifies price and stock server-side at order creation | M | — |
| FR-ORD-2 | Order state transitions follow the state machine; invalid transitions rejected and audited | M | ✔ `orders.state.changed` |
| FR-ORD-3 | An order may mix platform and vendor lines; each line snapshots listing type and vendor | M | — |
| FR-ORD-4 | Customer receives notification on placement, payment, dispatch, border crossing, delivery | M | ✔ consumes order/shipment events |
| FR-ORD-5 | Vendors see and action only the order lines belonging to them | M | ✔ `orders.placed` fan-out |
| FR-ORD-6 | Order cancellation releases stock and triggers refund when already paid | S | ✔ |

### 6.3 Payments
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-PAY-1 | Payment capture via provider adapter behind existing `PAYMENT_PROVIDER` port (sandbox first) | M | ✔ `payments.captured` / `payments.failed` |
| FR-PAY-2 | Every money amount stored as `numeric(12,2)` with explicit currency; no implicit conversion | M | — |
| FR-PAY-3 | Failed payment leaves order in a recoverable state; retry does not double-charge (idempotency key per attempt) | M | ✔ |
| FR-PAY-4 | Refund flow for cancelled paid orders | S | ✔ |

### 6.4 Shipping & cross-border
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-SHP-1 | Shipment lifecycle: `pending → booked → in_transit → at_border → customs_cleared → out_for_delivery → delivered | failed` | M | ✔ `shipments.status.changed` |
| FR-SHP-2 | Customer-visible tracking timeline built from shipment events (append-only history, not just latest status) | M | ✔ |
| FR-SHP-3 | Every cross-border shipment carries a `customsReference`; missing reference blocks `customs_cleared` | M | — |
| FR-SHP-4 | Courier adapter behind existing `SHIPPING_PROVIDER` port; provider callbacks map to shipment events | S | ✔ |

### 6.5 Vendors
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-VEN-1 | Vendor registration → admin approval workflow with status transitions audited | M | ✔ `vendors.status.changed` |
| FR-VEN-2 | Vendor dashboard: own listings, own order lines, fulfilment actions | M | — |
| FR-VEN-3 | Vendor settlement report (what HB owes each vendor per period) | C | ✔ projection from payment events |

### 6.6 Import requests / quotes (existing live service)
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-QTE-1 | Customer submits a one-off or recurring import request (personal or business) | S | ✔ `quotes.requested` |
| FR-QTE-2 | Ops respond with a transparent quote; SLA clock (≤ 48h) visible to admin | S | ✔ |
| FR-QTE-3 | Accepted quote converts to an order | C | ✔ |

### 6.7 Admin, audit & analytics
| ID | Requirement | Priority | Event |
|----|-------------|----------|-------|
| FR-ADM-1 | Admin oversight of catalog, vendors, orders, payments, shipments | M | — |
| FR-ADM-2 | Audit trail of privileged/state-changing actions is append-only and queryable | M | ✔ audit consumes all domain events |
| FR-ADM-3 | Operational dashboard (orders/day, GMV by listing type, border transit times) built as event projections | S | ✔ |

## 7. Non-functional requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | **Money integrity** — every service method touching money/inventory/order state has a unit test in the same PR (repo non-negotiable). |
| NFR-2 | **Contract integrity** — all API/UI types flow through `@hb/shared`; DTOs validated with class-validator. |
| NFR-3 | **Consistency over speed** — domain events are emitted transactionally with the state change (outbox pattern); no dual-write. At-least-once delivery; consumers idempotent. |
| NFR-4 | **Degraded-mode operation** — if the broker is down, synchronous commerce (browse/cart/checkout) keeps working; events queue in the outbox and drain on recovery. |
| NFR-5 | **SSR/SEO** — public catalog routes server-rendered; storefront usable on mobile. |
| NFR-6 | **Security & privacy** — POPIA (ZA) applies to personal data; secrets server-side only; audit log excludes sensitive payloads; refresh tokens hashed. |
| NFR-7 | **Schema discipline** — all changes via TypeORM migrations; `synchronize` off; Postgres enums stable-named. |
| NFR-8 | **Observability** — event flow inspectable (topic lag, dead-letter queue, consumer failures visible to admin). |
| NFR-9 | **Deployability** — API and web independently buildable/deployable; broker is a docker-compose service in dev; no Kubernetes required at this scale. |

## 8. Business rules (canonical, code-enforced where possible)

| ID | Rule | Enforcement |
|----|------|-------------|
| BR-1 | ZAR/NAD peg is 1:1 **as data**; every amount carries its currency column | Schema (`numeric(12,2)` + currency enum) |
| BR-2 | Vendor listings must have a vendor; platform listings must not | DB CHECK constraint |
| BR-3 | Customs is a first-class step: shipments record `fromCountry`, `toCountry`, `customsReference`; `at_border`/`customs_cleared` are mandatory states for cross-border legs | Enum + service logic |
| BR-4 | Order items snapshot price, listing type, and vendor at purchase time (later catalog edits never mutate history) | Entity design |
| BR-5 | Quote/contact requests answered within 24–48h (existing service promise) | SLA timer (FR-QTE-2) |
| BR-6 | Humans own production: no AI merge to `main`; every AI commit is trailer-tagged | Git hooks + factory log |

## 9. Assumptions & open questions (carry-over: verify before building)

From the legacy business-rules note — still **unverified**, must be confirmed with the business owner before the relevant card is played:

1. Final markup formula for platform-fulfilled listings.
2. Shipping-fee calculation rules (flat, weight-based, consolidated per border run?).
3. Tax handling (VAT treatment ZA→NA within SACU).
4. Customs documentation set required per product category.
5. Vendor settlement cadence and commission structure.
6. Whether recurring import requests need subscription-style billing.

## 10. Traceability

- **Judging criteria** → this document demonstrates *requirements gathering* and *AI-powered planning*; see `docs/ai-evidence/PITCH.md` (mono-repo) for the full mapping.
- **Architecture** → [[Target Architecture|TARGET-ARCHITECTURE.md]] and [[ADR-001 Event-Driven Backbone|ADR-001]] satisfy NFR-3/4/8 and every "Event ✔" requirement above.
- **Delivery** → [[Delivery Roadmap|ROADMAP.md]] sequences the FR groups into phases; each phase decomposes into Trello cards via `/spec-feature`.
