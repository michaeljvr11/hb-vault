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

## Snapshot (refreshed 2026-06-19, after VP-1)
- **27** commits (2026-06-11 → 2026-06-18); **21** carry the AI-authorship trailer (**78%**, climbing as the convention is followed)
- **+21 743 / −1 014** authored lines across **313** files (generated lockfiles excluded)
- **21** test specs (API 5 · Web 16)
- Guardrails fired: **8** prod-fence blocks logged (push-to-main ×6, force-push ×1, `gh pr merge` ×1 — all human-only) · **4** green PR gates (4 pass / 1 fail) · **4** lint fixups fed back to the agent
- Traceability: cards `mMFxZIKE` (auth, PR #1) + `L9fM1Gu7` (storefront, PR #4) + `Z1ADCLQm` (admin shell, PR #5) merged; `EXPtZCJL` (vendor portal shell, PR #7) in review. Trello flow: To Do 14 · In Progress 1 · In Review 2 · Done 3 · Documentation 7
