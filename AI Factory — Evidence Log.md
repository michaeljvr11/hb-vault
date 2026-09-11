---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log

**Snapshot (refreshed 2026-09-11)**

- **336** commits (2026-06-11 → 2026-09-11); **323** carry the AI-authorship trailer
- **162** test specs compiled
- Guardrails fired: **25** prod-fence blocks logged
- Traceability (latest): `83G6w4yZ` (SSR API calls routed to the internal service address; `x-forwarded-for` forwarded verbatim so the throttler keys on the real end user instead of collapsing every server-rendered visit onto one 120 req/min bucket) — branch `feat/83G6w4yZ-ssr-internal-api-routing`. Earlier: PD-1..PD-5 (5-card batch: product discounts & sale pricing — server-resolved discount window + effective price, order-line snapshot, vendor/admin discount UI, storefront sale badge) — branch `feat/UdeL4byD-product-discounts`, PR not yet opened. Earlier batch: `SZihvfYb` (5-card: product sizing). Earlier: `3QJYmybN` (6-card: storefront prelaunch polish). Earlier: `wokJ3PfW` (5-card: legal pages + consent durability).
