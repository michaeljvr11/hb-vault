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

**Snapshot (refreshed 2026-08-17)**

- **234** commits (2026-06-11 → 2026-08-17); **223** carry the AI-authorship trailer (**95%**)
- **+91,276 / −9,334** lines across **735** files
- **123** test specs compiled from the repo (API 54 · Web 69)
- Guardrails fired: **23** prod-fence blocks logged (push-to-protected, force-push, `gh pr merge`, merge/rebase on protected — all human-only) · **34** green PR gates · **201** lint issues fed back to agents
- Traceability (latest): `Ah6EZCOW` LSM-4/`9y69dIul` LSM-5/`TkbgWKL2` LSM-6 contact-page-and-inquiries (branch `feat/Ah6EZCOW-contact-page-and-inquiries`, PR open) — LSM-5 `POST /inquiries` endpoint (public, throttled, persist-first with guarded Resend notify), ContactInquiry entity + native Postgres enum + migration, DTO; LSM-4 `/contact` page (reactive form, ContactService, native HTML `<select>`/`<checkbox>`, WhatsApp CTA, prerendered); LSM-6 footer/nav wiring (Contact Support, About, Services, Register as Vendor links live), Home active-state bug fix, brand unified to "H&B E-Commerce" across nav/footer/auth/shells. Validation parity failures caught in review (API DTO caps vs form, Validators.email vs class-validator, @IsNotEmpty needed); WhatsApp green contrast regressed (white text 1.98:1, dark label 8.6:1 fix applied). Earlier: `kdro0zYC` LSM-1/`xhYmG5j8` LSM-2/`ahPpaEIK` LSM-3 landing-site migration (assets + SITE_IMAGES constants; tokens swapped `--hb-primary` `#015300`→`#2e7d32`, `--hb-secondary` `#964900`→`#f57c00`; contrast regressed, 2-round fix; `/about` + `/services` prerendered, teaser deleted + `/shop` cross-link, RenderMode.Prerender resolves SSR spec Q2). Earlier: `wygImWJb` TE-4 & `dwkfUoZ0` TE-5 order-paid notifications (domain event `order.paid` fans out N vendor + 1 platform + 1 customer emails; best-effort handlers; money formatting fixed). Earlier: `e7WlyfLC` TE-1/TE-2/TE-3 transactional email foundations (PR #53 merged). Earlier: `EDR-1` Earnings date-range picker (PR #52 merged). See repo `REPORT.md` for complete per-card/per-PR history.
