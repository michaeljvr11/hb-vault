# Delivery Roadmap — HB Platform

**Status:** Draft v0.1 — for review · **Date:** 2026-07-07
**Companions:** [[BRS — Business Requirements|BRS]] · [[ADR-001 Event-Driven Backbone|ADR-001]] · [[Target Architecture|Target architecture]]

Each phase decomposes into Trello cards via `/spec-feature` (one card = one branch =
one PR). Phases are sequenced so that **every phase ends demo-able** — important for a
hackathon where the judging moment may land mid-phase.

## Phase 0 — Foundation ✅ (done, on `main`)

Monorepo + shared contracts, auth, catalog + search, storefront (home, discovery,
PDP), server-side cart, checkout, order creation + state machine, stub payment
capture, security-hardening audit, AI-evidence pipeline.

## Phase 1 — Close the transactional loop (~1 sprint)

*Goal: a customer can complete a real end-to-end purchase; vendors see their orders.*

| # | Slice | BRS refs |
|---|-------|----------|
| 1.1 | Payment provider adapter (sandbox: Payfast or Paystack) behind `PAYMENT_PROVIDER`; webhook → payment status; idempotent capture | FR-PAY-1/3 |
| 1.2 | Order fulfilment actions (confirm → processing → shipped) for admin + vendor, state machine enforced | FR-ORD-2/5 |
| 1.3 | Vendor order views (own lines only) | FR-VEN-2 |
| 1.4 | Order history + detail for customers | FR-ORD-* |
| 1.5 | Cancellation + stock release (refund stubbed until 1.1 lands) | FR-ORD-6 |

**Exit demo:** browse → pay (sandbox) → vendor fulfils → customer sees status.

## Phase 2 — Event backbone (ADR-001) (~1 sprint)

*Goal: the async seam exists and three real consumers run on it.*

| # | Slice | BRS refs |
|---|-------|----------|
| 2.1 | `@hb/shared` event envelope + `events` module: outbox entity/migration, relay, `EVENT_BUS` port with in-memory + `LISTEN/NOTIFY` adapters | NFR-3 |
| 2.2 | Producers: orders, payments, vendors, catalog emit via outbox | FR-ORD-2, FR-PAY-1, FR-VEN-1, FR-CAT-2 |
| 2.3 | Kafka (KRaft) in docker-compose + kafkajs adapter + DLQ convention | NFR-4/8/9 |
| 2.4 | Consumer: notifications (order placed / paid / shipped emails via mail module) | FR-ORD-4 |
| 2.5 | Consumer: search indexer on catalog events | FR-CAT-2 |
| 2.6 | Consumer: audit projection (+ parity check against direct writes) | FR-ADM-2 |

**Exit demo:** place an order → watch the event fan out (emails, audit, dashboard
counter) — then kill the Kafka container, place another order, restart, and show it
drain. That degraded-mode demo is the architecture pitch in 60 seconds.

## Phase 3 — Cross-border visibility (~1 sprint)

*Goal: the differentiator — the border leg is transparent.*

| # | Slice | BRS refs |
|---|-------|----------|
| 3.1 | Shipment creation on dispatch; `shipments.status.changed` events; `shipment_events` timeline table | FR-SHP-1/2 |
| 3.2 | Customer tracking page: timeline incl. `at_border` / `customs_cleared` milestones | FR-SHP-2, BO-3 |
| 3.3 | `customsReference` enforcement + admin customs worklist | FR-SHP-3 |
| 3.4 | Border/customs notifications to customer | FR-ORD-4 |
| 3.5 | (Stretch) courier adapter with callback → event mapping | FR-SHP-4 |

**Exit demo:** admin advances a shipment through the border; customer timeline and
notifications update live.

## Phase 4 — Two-sided growth (post-hackathon horizon)

Quote/import-request digitization with 48h SLA tracking (FR-QTE-*), vendor settlement
projections (FR-VEN-3), operational analytics from event replay (FR-ADM-3), returns,
KYC deepening. Sequenced after real usage data exists — deliberately not detailed yet.

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| Kafka setup burns hackathon time | ADR-001 D1/D2 split: outbox works on `LISTEN/NOTIFY`; the broker is a swap, not a prerequisite. Phase 2.3 is skippable without blocking 2.4–2.6. |
| Payment sandbox onboarding delays (account approval) | Start provider signup immediately (calendar item, not a card); stub remains the fallback demo path. |
| Two devs, parallel branches, shared contracts | `@hb/shared` changes land in their own small PRs first; check `origin/main` before starting any slice. |
| Open business questions block slices (BRS §9) | Each Phase-3+/quote card carries its "needs verification" question; `/spec-feature` resolves it with the business owner before implementation. |
| Scope creep toward microservices | ADR-001 explicitly scopes events to *module seams inside one monolith*; any service split needs a new ADR. |

## Suggested judging narrative (ties BO-6 together)

1. **Requirements** — BRS with numbered, code-traceable requirements (this folder).
2. **Decision discipline** — ADR-001: option analysis, honest costs, reversibility.
3. **Live demo** — Phase 2 exit demo (event fan-out + degraded mode).
4. **Evidence** — `npm run evidence` → AI-authored %, guardrail log, card→PR trace.
