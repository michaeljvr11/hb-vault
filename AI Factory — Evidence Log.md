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

**Snapshot (refreshed 2026-08-15)**

- **221** commits (2026-06-11 → 2026-08-15); **210** carry the AI-authorship trailer (**95%**)
- **+87,529 / −8,721** lines across **668** files
- **118** test specs compiled from the repo (API 51 · Web 67)
- Guardrails fired: **23** prod-fence blocks logged (push-to-protected, force-push, `gh pr merge`, merge/rebase on protected — all human-only) · **34** green PR gates · **201** lint issues fed back to agents
- Traceability (latest): `kdro0zYC` LSM-1/`xhYmG5j8` LSM-2/`ahPpaEIK` LSM-3 landing-site migration (branch `feat/kdro0zYC-landing-site-migration`, PR open) — assets + `SITE_IMAGES` constants module; tokens swapped (`--hb-primary` `#015300`→`#2e7d32`, `--hb-secondary` `#964900`→`#f57c00`), pill-button radius scoped to primary/CTA, `--hb-primary-container` hand-picked to `#43a047`; contrast regression caught (2-round fix: fill text swaps + `--hb-secondary` reclassified as fills-only); `/about` + `/services` prerendered with `RenderMode.Prerender` entries (resolves [[Public Storefront & SSR]] open Q2), launch-neutral copy, teaser deleted + `/shop` cross-link on services. Earlier: `wygImWJb` TE-4 & `dwkfUoZ0` TE-5 order-paid notifications (branch `feat/wygImWJb-order-paid-notifications`, PR ready) — domain event `order.paid` fans out N vendor + 1 platform + 1 customer emails per confirmed-paid order; vendor emails show only their lines, platform shows all lines + total, customer shows all lines + total; each send wrapped in best-effort `safely()` handler (no retry queue per v1 trade-off); money formatting fixed (`.toFixed(2)`) to preserve cents; TE-5 customer confirmation backfilled as follow-up to TE-4's research. Earlier: `e7WlyfLC` TE-1/TE-2/TE-3 transactional email foundations (branch `feat/e7WlyfLC-transactional-email-foundations`, PR #53 merged) — branded email template, platform admin settings + API + UI, vendor notification override, multi-recipient support, transport hardening. Earlier: `EDR-1` Earnings date-range picker (PR #52 merged). See repo `REPORT.md` for complete per-card/per-PR history.