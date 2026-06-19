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

## Snapshot (refreshed 2026-06-19, after AP-7 lifecycle hardening / PR #9)
- **33** commits (2026-06-11 → 2026-06-19); **27** carry the AI-authorship trailer (**82%**)
- **+23 193 / −1 201** authored lines across **318** files (generated lockfiles excluded)
- **22** test specs (API 5 · Web 17)
- Guardrails fired: **9** prod-fence blocks logged (push-to-protected ×7, force-push ×1, `gh pr merge` ×1 — all human-only) · PR gate pass/fail tracked · agent edits logged by the lint hook
- Traceability: cards `mMFxZIKE` (auth, PR #1) + `L9fM1Gu7` (storefront, PR #4) + `Z1ADCLQm` (admin shell, PR #5) + `EXPtZCJL` (vendor portal shell, PR #7) + `DD6Z0NUW` (vendor approvals UI, PR #8) merged; `RLiauFte` (vendor status lifecycle server-side enforcement, PR #9) in review. Slice 7 now complete end-to-end (UI PR #8 + server guard PR #9).
