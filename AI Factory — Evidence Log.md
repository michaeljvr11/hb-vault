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

**Snapshot (refreshed 2026-08-27)**

- **274** commits (2026-06-11 → 2026-08-27); **263** carry the AI-authorship trailer (**96%**)
- **+110 288 / −11 509** lines across **842** files
- **152** test specs (API 72 · Web 80)
- Guardrails fired: **23** prod-fence blocks logged (push names a protected branch ×17; force-push ×1; `gh pr merge` ×2; merge/rebase on protected ×2; bare push from protected ×1 — all human-only) · **36** green PR gates · **281** lint issues fed back to agents
- Traceability (latest): `wokJ3PfW` et al. PR open (5-card batch: the last two legal pages — export & customs, vendor agreement — footer wiring, a durable `users.termsAcceptedAt` consent column, and a Google-OAuth consent interstitial) — branch `feat/wokJ3PfW-legal-pages-and-consent-durability`. Earlier batch: `QQKjOiEH` (4-card batch: legal policy pages — privacy, cookies, terms, shipping, returns — plus signup consent capture and audit record). Earlier batch: `DwnyCLnX` (6-card batch: configurable shipping fee, route+currency keyed, 8-row sets, per-product override sparse/mutable, MAX across lines, checkout parity spec) — branch `feat/DwnyCLnX-configurable-shipping-fee`.