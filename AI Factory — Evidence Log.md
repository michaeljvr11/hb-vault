---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log

**Snapshot (refreshed 2026-09-14)**

- **362** commits (2026-06-11 → 2026-09-14); **341** carry the AI-authorship trailer (94%)
- **167** test specs compiled (API 78 · Web 89)
- **+135,000 / −17,591 lines** across **941 files**
- Guardrails fired: **27** prod-fence blocks · **42** green PR gates · **326** lint issues fed back
- Traceability (latest): `YYjbfc65` (SEC-5, SEC-7 bundled; SEC-5 nonce-based CSP for HTML origin + SEC-7 access token to in-memory storage closing audit M4 completely, plus SEC-6 Angular patch guardrail + SEC-8 Meilisearch scoped keys) — branch `feat/YYjbfc65-sec-cleanup-batch`. Earlier: `BGykPFS7` (SEC-1..SEC-4, 4-card batch: static security headers on the HTML origin at the Caddy edge, env-driven basic-auth + `noindex` gate for non-production, gitleaks blocking + Dependabot + reporting-only `npm audit` in CI, CORS fails closed in production) — branch `feat/BGykPFS7-security-guards-batch`. Earlier: `83G6w4yZ` (SSR API calls routed to the internal service address; `x-forwarded-for` forwarded verbatim so the throttler keys on the real end user instead of collapsing every server-rendered visit onto one 120 req/min bucket) — branch `feat/83G6w4yZ-ssr-internal-api-routing`. Earlier: PD-1..PD-5 (5-card batch: product discounts & sale pricing) — branch `feat/UdeL4byD-product-discounts`. Earlier batch: `SZihvfYb` (5-card: product sizing). Earlier: `3QJYmybN` (6-card: storefront prelaunch polish). Earlier: `wokJ3PfW` (5-card: legal pages + consent durability).
