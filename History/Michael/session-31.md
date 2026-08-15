# Session 31 — TE-4/TE-5: Order-paid notifications & customer confirmation

**Date:** 2026-08-15 · **Cards:** [TE-4](https://trello.com/c/wygImWJb) / [TE-5](https://trello.com/c/dwkfUoZ0) · **Branch:** `feat/wygImWJb-order-paid-notifications` · **Status:** Ready for PR

- Shipped: Domain event `OrderEvents.PAID` emitted at `capturePayment()` on confirmed-paid transition. Listener (`OrderNotificationsListener`) dispatches N vendor + 1 platform + 1 customer email per paid order; vendor emails show only that vendor's lines, platform email shows all lines + total, customer email shows all lines + total. Each send wrapped in `safely()` catch-log-swallow (best-effort, no retry queue per v1 trade-off). Review fix: money formatting (`.toFixed(2)`) applied at five render sites in `mail.service.ts` to preserve cents from pg `numeric(12,2)`. TE-5 (customer confirmation) backfilled as follow-up to TE-4's research.
- Decisions: Best-effort/no-safety-net aligns with `SearchIndexerService` precedent; no durable queue (noted for future). Customer notification (TE-5) closed gap from Open Question 1, confirmed as separate follow-up rather than in-scope of TE-4. Money formatting at render time, not DB layer (audit preservation).
- Tests: 51 suites / 693 tests pass, API 535/535, lint clean, full build clean.
- Follow-ups: none.
