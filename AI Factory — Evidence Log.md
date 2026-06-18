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
| Design | Stitch MCP → design-to-code agent |
| Implementation | backend / frontend engineers (sonnet) |
| Testing | test-engineer + PR-gate hook |
| Review | code-reviewer (opus) |
| Docs | docs-writer (haiku) + the evidence report |
| Safety | enforced hooks — prod fence, secrets fence |

## Snapshot — 2026-06-18
_Source: `npm run evidence`. Refresh each cycle; the repo `REPORT.md` is the source of truth._

- **9** commits (2026-06-11 → 2026-06-18); **5** carry the AI-authorship trailer (**56%**, climbing as the convention is followed)
- **+11 273 / −637** authored lines across **241** files (generated lockfiles excluded)
- **8** test specs (API 1 · Web 7)
- Guardrails fired: **3** prod-fence blocks logged (push-to-main, force-push, `gh pr merge` — all human-only)
- Traceability: card `mMFxZIKE` → `feat/mMFxZIKE-login-registration-pages` → PR #1 (open)
- Trello flow: To Do 6 · In Progress 0 · In Review 1 · Done 0 · Documentation 7
