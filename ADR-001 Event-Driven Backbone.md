# ADR-001: Event-driven backbone (transactional outbox first, Kafka as the transport)

**Status:** Proposed — pending human approval
**Date:** 2026-07-07
**Deciders:** Josh Da Silva, Michael (+ AI consultant draft)
**Supersedes:** the original skeleton guardrail "Microservices/K8s/brokers — single modular monolith" (README §Not included). That guardrail was correct for the skeleton phase; this ADR documents the deliberate, scoped exception now that the transactional core exists.

## Context

The platform has reached the point where several requirements are *inherently
asynchronous fan-out* — one state change, many interested parties:

- An order placement must notify the customer, alert each vendor with lines in the
  order, write the audit trail, and update admin dashboards (FR-ORD-4/5, FR-ADM-2/3).
- A shipment crossing the border must append to the customer-visible tracking
  timeline and feed transit-time analytics (FR-SHP-2, FR-ADM-3).
- A catalog edit must re-index search (FR-CAT-2).

Doing this synchronously inside request handlers couples every module to every other,
makes checkout latency hostage to email delivery, and loses history (we store *latest
status*, not *what happened*). Doing it with fire-and-forget calls creates the
dual-write problem: DB commit succeeds, notification is lost (or vice versa).

Separately, this is a Tech Accelerator project: the architecture decision itself is a
judged artifact. The decision below is defensible on engineering merit alone; demo
value is a tiebreaker, not the driver.

## Decision

**Adopt event-driven architecture in two decoupled decisions:**

### D1 — Domain events via transactional outbox (the important decision)
- Define a versioned event envelope in `@hb/shared` (`eventId`, `type`, `aggregateId`,
  `occurredAt`, `version`, `payload`) — the event contract is shared exactly like DTOs.
- Services that change money/inventory/order/vendor/shipment state write the event to
  an `outbox` table **in the same DB transaction** as the state change. No dual-write,
  ever.
- A relay (NestJS scheduled worker in the same process) drains the outbox to the
  transport with at-least-once semantics; consumers are idempotent (keyed on `eventId`).
- **The API remains one modular monolith.** Producers and consumers are NestJS modules;
  events are the seam between modules, not a service boundary.

### D2 — Kafka as the transport (the swappable decision)
- Single-node **Kafka in KRaft mode** added to `docker-compose.yml` (no ZooKeeper).
  Redpanda is the fallback if the Kafka container proves heavy on dev machines — it is
  Kafka-API-compatible, so nothing above the client changes.
- Topics: `hb.orders`, `hb.payments`, `hb.shipments`, `hb.catalog`, `hb.vendors`,
  `hb.quotes` — partition key = aggregate ID (per-aggregate ordering guaranteed).
- Client: `kafkajs` behind an `EVENT_BUS` injection token (same port-and-adapter
  pattern as `PAYMENT_PROVIDER` / `SHIPPING_PROVIDER`).
- Dead-letter topic per consumer group; lag/failures surfaced in the admin portal (NFR-8).

**Degraded mode (NFR-4):** if the broker is down, checkout and browsing still work —
events accumulate in the outbox and drain on recovery. The broker is additive
infrastructure, never a synchronous dependency of the request path.

## Options considered

| Option | Verdict | Reasoning |
|---|---|---|
| **A. Status quo** — synchronous calls in request handlers | Rejected | Couples modules, checkout latency inherits email/courier latency, no history, dual-write bugs already lurking in notification logic. |
| **B. Outbox + Postgres `LISTEN/NOTIFY`** — no new infra | Viable, not chosen as target | Correct and cheapest; but no replay, no consumer groups, notifications dropped if no listener connected. Acceptable *interim* transport — D1 works with this today, which is why D1 ships first. |
| **C. BullMQ (Redis)** | Rejected as backbone | Job queue, not an event log: point-to-point, no fan-out to N consumers without N queues, no replay. Right tool for *jobs* (image resize), wrong shape for *domain events*. |
| **D. RabbitMQ** | Runner-up | Mature fan-out, lighter ops than Kafka. Loses on: no native replay/retention (the shipment timeline and audit projection want a log), and weaker analytics story. |
| **E. Kafka / Redpanda** | **Chosen (D2)** | Durable replayable log fits audit-as-projection and tracking-timeline requirements; consumer groups give clean per-concern consumers; partition-per-aggregate ordering matches the order/shipment state machines. Ops cost contained by single-node KRaft in compose + degraded-mode rule. |

## Honest costs (recorded so nobody is surprised)

1. **Operational surface** — one more stateful service in dev and prod. Mitigation: compose-managed, single node, monolith survives without it.
2. **Idempotency discipline** — at-least-once delivery means every consumer must tolerate duplicates. Mitigation: `processed_events` table keyed on `eventId`; enforced by a shared consumer base class + unit tests (NFR-1 applies).
3. **Schema governance** — event payloads are contracts. Mitigation: they live in `@hb/shared` with explicit `version`; additive-only changes.
4. **Team learning curve** — partitioning/rebalancing concepts. Mitigation: one topic naming + keying convention documented in TARGET-ARCHITECTURE.md; nobody tunes partitions at this scale (default 3).
5. **Hackathon time risk** — D2 is deferrable. If time runs short, ship D1 on `LISTEN/NOTIFY` (option B) and swap the transport later — the `EVENT_BUS` port makes that a one-adapter change, exactly like the payment/shipping ports.

## Consequences

- New modules: `events` (envelope, outbox entity, relay, `EVENT_BUS` port + adapters) and consumer wiring in `mail`, `search`, `audit`, `admin`.
- New migrations: `outbox`, `processed_events`, `shipment_events` (timeline).
- `docker-compose.yml` gains a `kafka` service (dev). CI runs consumers against the in-memory/`LISTEN/NOTIFY` adapter — tests never require a broker.
- The audit module gains a second write path (event projection) — existing direct writes remain until parity, then are removed.
- README §"Not included" updated to reference this ADR.
