---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log_Source: `npm run evidence`. Refresh each cycle; the repo `REPORT.md` is the source of truth._

Snapshot (refreshed 2026-07-20, after PR #35 Public Vendor Profile)
- **142** commits (2026-06-11 → 2026-07-20); **132** carry the AI-authorship trailer (**93%**)
- **87** test specs compiled from the repo (API 33 · Web 54)
- Guardrails fired: **17** prod-fence blocks logged (push-to-protected, force-push, `gh pr merge`, merge/rebase on protected — all human-only) · **19** green PR gates · **116** lint issues fed back to agents
- Traceability (latest): PR #35 Public Vendor Profile (card UG5UFyxy #40) — public `/vendors/:id` page, design + web UI, slice 5 of storefront epic; 1 commit; test/review SHIP. Earlier: PR #34 Analytics & Reporting batch (cards LKIGKUVM/FRtcp2dg/p0kVZMii/UC508IB6/ezkaNtbZ) — @hb/shared analytics contracts + event ingestion + storefront instrumentation + GA4 consent-gating + admin funnel/revenue dashboards + vendor analytics; 10 commits; test/review SHIP. Earlier: PR #33 Customer Profile batch (cards eq7ybOyS/U9C8764Y/1StR8MKR/xkD93Anr/uGvvSURk) — user profile endpoints + addresses CRUD + order history + address book UI; 7 commits; test/review SHIP. Earlier: Product Search Engine batch (cards `sI1Vaxp0` #45 SearchModule + Meilisearch, `jphU7kkC` #48 ProductSearchService, `ig9t1BQQ` #49 synonyms load, `DyK4yTyp` #52 synonyms admin CRUD + reload, `MkRjnkvv` #53 synonyms admin UI); admin portal/vendor portals/storefront+ SSR/auth epics merged in full; see repo `REPORT.md` for complete per-PR history.