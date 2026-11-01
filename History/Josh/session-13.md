---
operator: Josh
date: 2026-07-07
session: 13
tags: [architecture, business-docs, kafka, roadmap, trello, retroactive]
---

# Session 13 — Architecture consult: BRS, Kafka ADR, roadmap + board order

> Logged retroactively on 2026-11-01 from chat history. No code or PR. This was planning only.

## What we worked on
Claude acted as a consultant on next steps for the Tech Accelerator hackathon. The question was
whether to add Kafka, and the agreed first step was a proper BRS before any build.

## Deliverables
Four documents. They were first drafted into the mono-repo's `docs/` (left untracked) and moved
into this vault on 2026-11-01 as [[BRS — Business Requirements]], [[Delivery Roadmap]],
[[ADR-001 Event-Driven Backbone]] and [[Target Architecture]]:
- `docs/business/BRS.md`: the Business Requirements Specification. Objectives, stakeholders, the
  verified as-is state, about 30 numbered functional requirements (MoSCoW, each flagged if it needs
  events), NFRs, business rules, and the open questions carried over.
- `docs/architecture/ADR-001-event-driven-backbone.md`: the Kafka decision record, with five options analysed.
- `docs/architecture/TARGET-ARCHITECTURE.md`: as-is and target diagrams (mermaid), event taxonomy
  with topic/keying conventions, the consistency and failure model, and a test strategy that keeps
  brokers out of CI.
- `docs/business/ROADMAP.md`: four demo-able phases. Close the transactional loop, then the event
  backbone, then cross-border tracking, then two-sided growth. Includes a risk table and a judging narrative.

## Key decisions
- **Kafka is two decisions, not one.** The decision that matters is event-driven architecture via a
  **transactional outbox**: events are written in the same Postgres transaction as the state change,
  which removes dual-write bugs with no new infrastructure. Kafka is only the swappable transport
  behind an `EVENT_BUS` token, the same port/adapter pattern as payments and shipping.
- This overturns the README's original "no brokers, single modular monolith" guardrail, so it has to
  be argued in an ADR, not slipped in. The monolith stays; events are module seams, not a microservice split.
- Demo idea: kill the broker mid-demo, place an order, restart it, and watch the outbox drain.

## Board order (same day)
Read the Trello "To Do" queue top to bottom: agentic environment, scaffold, hosting, the vendor portal
epic, VP-4 (partly blocked on checkout), the public vendor profile, pagination and sort, then the
Meilisearch search slices 1 → 6 and synonyms 5a/5b/5c. The canonical-product/buy-box card is
explicitly deferred out of v1. Noted that the synonyms admin slices (5b/5c) depend on 5a, not on the
search API slice, so they can run in parallel. Then started the app (Postgres, API on 3000, web on 4200).

## Follow-ups
- The four docs are Draft v0.1. Their as-is sections predate analytics, discounts, search parity and
  order tracking, so they need a refresh.
- ADR-001 is still **Proposed**. Analytics (session 10) deliberately built direct ingestion with event
  names aligned to it.
