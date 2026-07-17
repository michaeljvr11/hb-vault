---
type: process
tags:
  - ai-factory
  - evidence
  - hackathon
---

# AI Factory — Evidence Log

The auditable record of how AI tools built HB. This note is the **vault-side mirror** of
the machine-generated report at `docs/ai-evidence/REPORT.md` in the repo. Refresh the
snapshot below after each `/ship-card` cycle (run `npm run evidence` first).

> Why this exists: the hackathon's R10 000 AI-tooling award rewards *intentional, effective
> use of AI throughout development, with evidence of value*. We measure it instead of
> asserting it. See [[HB Domain Model]] for the product these agents are building.

## How evidence is captured (continuously)
1. **AI-authorship trailer** — every AI commit ends `Co-Authored-By: Claude <noreply@anthropic.com>`.
   Audit it with `git log --all --grep="Co-Authored-By.*Claude"`.
2. **Guardrail telemetry** — the prod-fence / PR-gate / lint hooks append every firing to
   `.claude/factory-log.jsonl`. The safety story, with timestamps.
3. **Evidence generator** — `npm run evidence` compiles git + telemetry + Trello + GitHub
   into `docs/ai-evidence/REPORT.md`. Re-runnable; always current.

## Lifecycle coverage — an AI agent per phase
| Phase | AI driver |
|---|---|
| Requirements | Obsidian MCP + these business-rule notes |
| Planning | `/ship-card` plan step + Trello board |
| Design | Claude Design → design-to-code agent |
| Implementation | backend / frontend engineers (sonnet) |
| Testing | test-engineer + PR-gate hook |
| Review | code-reviewer (opus) |
| Docs | docs-writer (haiku) + the evidence report |
| Safety | enforced hooks — prod fence, secrets fence

_Source: `npm run evidence`. Refresh each cycle; the repo `REPORT.md` is the source of truth._

Snapshot (refreshed 2026-07-17, after PR #34 Analytics & Reporting batch)
- **139** commits (2026-06-11 → 2026-07-17); **129** carry the AI-authorship trailer (**93%**)
- **80** test specs compiled from the repo
- Guardrails fired: **17** prod-fence blocks logged (push-to-protected, force-push, `gh pr merge`, merge/rebase on protected — all human-only) · PR gate runs (pass) · lint issues fed back to agents
- Traceability (latest): PR #34 Analytics & Reporting batch (cards LKIGKUVM/FRtcp2dg/p0kVZMii/UC508IB6/ezkaNtbZ) — @hb/shared analytics contracts + event ingestion + storefront instrumentation + GA4 consent-gating + admin funnel/revenue dashboards + vendor analytics; 10 commits; test/review SHIP. Earlier: PR #33 Customer Profile batch (cards eq7ybOyS/U9C8764Y/1StR8MKR/xkD93Anr/uGvvSURk) — user profile endpoints + addresses CRUD + order history + address book UI; 7 commits; test/review SHIP. Earlier: Product Search Engine batch (cards `sI1Vaxp0` #45 SearchModule + Meilisearch, `jphU7kkC` #48 ProductSearchService, `ig9t1BQQ` #49 synonyms load, `DyK4yTyp` #52 synonyms admin CRUD + reload, `MkRjnkvv` #53 synonyms admin UI); admin portal/vendor portals/storefront+ SSR/auth epics merged in full; see repo `REPORT.md` for complete per-PR history.
