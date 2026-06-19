---
type: process
tags: [ai-factory, evidence, hackathon]
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

## Snapshot (refreshed 2026-06-19, after VP-5 vendor onboarding / PR #12)
- **41** commits (2026-06-11 → 2026-06-19); **36** carry the AI-authorship trailer (**88%**)
- **+28 176 / −1 541** authored lines across **336** files (generated lockfiles excluded)
- **24** test specs (API 5 · Web 19)
- Guardrails fired: **13** prod-fence blocks logged (push-to-protected ×10, force-push ×1, `gh pr merge` ×2 — all human-only) · PR gate pass/fail tracked · agent edits logged by the lint hook
- Traceability: cards `mMFxZIKE` (auth, PR #1) + `L9fM1Gu7` (storefront, PR #4) + `Z1ADCLQm` (admin shell, PR #5) + `EXPtZCJL` (vendor portal shell, PR #7) + `DD6Z0NUW` (vendor approvals UI, PR #8) + `RLiauFte` (vendor status lifecycle, PR #9) + `IhTVdNYP` (admin catalog, PR #10) merged; `N8P6OPPm` (admin user management, PR #11) + `1fqtc2JF` (vendor onboarding VP-5, PR #12) in review.