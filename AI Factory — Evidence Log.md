---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log
_Source: `npm run evidence`. Refresh each cycle; the repo `REPORT.md` is the source of truth._

**Snapshot (refreshed 2026-08-13)**

- **204** commits (2026-06-11 → 2026-08-13); **193** carry the AI-authorship trailer (**95%**)
- **+84,300 / −8,342** lines across **647** files
- **109** test specs compiled from the repo
- Guardrails fired: **21** prod-fence blocks logged (push-to-protected, force-push, `gh pr merge`, merge/rebase on protected — all human-only) · **34** green PR gates · **179** lint issues fed back to agents
- Traceability (latest): `EDR-1` Earnings date-range picker (branch `feat/EDR-1-earnings-date-range-picker`, PR about to open) — all-time preset + custom date picker on vendor/admin earnings screens, shared selector component, epoch sentinel, index migration on `orders.createdAt` / `order_items.vendorId`, client-side settlement pagination, `@IsNotFutureDate()` guard. Earlier: `IP59Crue` / `7sclIgtI` / `BP5q32o4` Storefront UI cleanup batch (UIC-1/2/3, PR #51 merged). Earlier: `rtBV85cQ` Product wishlist (WL-1–5) — signal-backed `wishlist_items` table + API + UI. See repo `REPORT.md` for complete per-card/per-PR history.