---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log
# AI Factory — Evidence Log
_Source: `npm run evidence`. Refresh each cycle; the repo `REPORT.md` is the source of truth._

**Snapshot (refreshed 2026-08-26)**

- **268** commits (2026-06-11 → 2026-08-26); **257** carry the AI-authorship trailer (**96%**)
- **+108 532 / −11 358** lines across **831** files
- **149** test specs (API 72 · Web 77)
- Guardrails fired: **23** prod-fence blocks logged (push names a protected branch ×17; force-push ×1; `gh pr merge` ×2; merge/rebase on protected ×2; bare push from protected ×1 — all human-only) · **36** green PR gates · **278** lint issues fed back to agents
- Traceability (latest): `QQKjOiEH` et al. PR open (4-card batch: legal policy pages — privacy, cookies, terms, shipping, returns — plus signup consent capture and audit record) — branch `feat/QQKjOiEH-legal-policy-pages`. Earlier batch: `DwnyCLnX` (6-card batch: configurable shipping fee, route+currency keyed, 8-row sets, per-product override sparse/mutable, MAX across lines, checkout parity spec) — branch `feat/DwnyCLnX-configurable-shipping-fee`.