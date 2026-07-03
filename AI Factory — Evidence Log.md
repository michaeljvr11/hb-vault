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

Snapshot (refreshed 2026-07-02, after b4VoyjRu + 6qlkwk75 discovery API + storefront UI / PR #26)
- **85** commits (2026-06-11 → 2026-07-02); **76** carry the AI-authorship trailer (**89%**)
- **+44 425 / −3 371** authored lines across **423** files (generated lockfiles excluded)
- **49** test specs (API **12** · Web **37**)
- Guardrails fired: **16** prod-fence blocks logged (push-to-protected ×12, force-push ×1, `gh pr merge` ×2, merge/rebase on protected ×1 — all human-only) · **12** PR gate runs (pass) · **65** lint issues fed back to agent
- Traceability: cards `mMFxZIKE` (auth, PR #1) + `L9fM1Gu7` (storefront, PR #4) + `Z1ADCLQm` (admin shell, PR #5) + `EXPtZCJL` (vendor portal shell, PR #7) + `DD6Z0NUW` (vendor approvals UI, PR #8) + `RLiauFte` (vendor status lifecycle, PR #9) + `IhTVdNYP` (admin catalog, PR #10) + `N8P6OPPm` (admin user management, PR #11) merged; `1fqtc2JF` (vendor onboarding VP-5, PR #12) + `oWqIV57s` (vendor dashboard VP-2, PR #15) + `xEOk5i0M` (vendor products VP-3, PR #14) + `sw8qYBV1` (admin orders AP-10, PR #16) + `7oPc1cnk` (admin audit log AP-11, PR #17) + `kYbVZyqV` (public storefront + SSR PS-1, PR #19) + `eOI6gKpN` (returnUrl guard capture PS-2, PR #20) + `vs3y5tDJ` (public vendor profile PS-3, PR #21) merged; `EFXHykuT` (auth-aware storefront nav PS-4, PR #22, completes the Public Storefront & SSR epic) + `c2o6xfZs` (approved-vendor product filter, PR #23) + `Sm1kSNO8` (safe category delete — block in-use / parent categories, PR #24) merged; `b4VoyjRu` (discovery filters categoryId/q/vendorId + unified suggest) + `6qlkwk75` (storefront browse+search UI + product-detail) shipped together on `fable-storefront-oneshot` (PR #26) **awaiting human merge**.
