---
operator: Josh
date: 2026-07-17
session: 15
tags: [ai-factory, ship-batch, clarify, token-usage, retroactive]
---

# Session 15 — AI Factory upgrades: /ship-batch, CLARIFY, steering audit, token tracking

> Logged retroactively on 2026-11-01 from chat history and PR #31.

## What we worked on
Improving the AI Factory itself (no Trello card). Everything shipped in one review PR:
[PR #31](https://github.com/michaeljvr11/hb-mono-repo/pull/31), merged.

## What changed
- **Consistency PULL for `/spec-feature`.** It never re-synced the repo and vault before writing a
  spec, unlike `/ship-card`. Now it does.
- **Two report-only skills.** `align-steering-docs` checks CLAUDE.md files, commands, agents and
  skills against the real project state. `service-extraction-analysis` ranks candidates to pull out
  of the monolith using git churn, coupling and module growth. Both feed new sections in the evidence report.
- **`/ship-batch`.** Formalises the card-bundling done ad hoc in PRs #26/#27/#28 into a golden-path
  command. It has the same stages as `/ship-card`, decides parallel vs sequential, and supports
  one-shot large builds offloaded to Fable.
- **CLARIFY step** between PLAN and IMPLEMENT in both `/ship-card` and `/ship-batch`. When
  requirements are unclear, the agent stops and asks the developer instead of assuming. It only asks
  when it has to.
- **Ponytail minimalism ladder**, adapted from `DietrichGebert/ponytail`: "does this need to exist →
  reuse → stdlib → existing dep → shortest diff". It was copied into the repo (no plugin install) and
  added to `backend-engineer`, `frontend-engineer` and an advisory, non-blocking over-engineering pass
  in `code-reviewer`. It adds a `// ponytail: <ceiling>, <upgrade trigger>` comment convention.
  Non-negotiables (validation, money/order tests, authz) are explicitly outside the ladder.
- **Per-agent token usage.** A new `SubagentStop` hook (`log-agent-tokens.js`) logs each specialist's
  input/output/cache tokens to `factory-log.jsonl`. The evidence generator renders a "Token usage by
  agent" table in `REPORT.md` and `dashboard.html`.

## Key decisions
- The two factory branches and the Ponytail work were folded into a single PR at Josh's request.
  Ponytail's separate PR #32 was closed.
- The point of token tracking is to **measure** whether changes like Ponytail reduce spend: snapshot
  `npm run evidence` before and after.
- Synthetic test entries were removed from `factory-log.jsonl` before commit. It is an audit log and
  must not ship fake data.

## Follow-ups
- Compare backend and frontend agent token totals across batches to judge the effect of Ponytail.
