---
operator: Josh
date: 2026-06-18
session: 1
tags: [ai-factory, ship-card, design-system]
---

# Session 1 — AI Factory loop + Claude Design integration

## What we worked on
Improving the AI Factory itself (not a Trello card): tightening the `/ship-card`
loop and verifying Michael's Stitch → Claude Design migration.

## What changed
- **`/ship-card` loop (`.claude/commands/ship-card.md`)** — IMPLEMENT is now a
  per-slice loop:
  1. PULL latest for **both** repos (project + Obsidian vault), then re-read the
     relevant spec notes to catch changes since CONTEXT.
  2. BUILD via specialists.
  3. COMMIT CODE to the project feature branch.
  4. DOCUMENT — docs-writer appends Implementation Notes **and** records a session
     log under `History/<Name>/session-<n>.md`.
  5. COMMIT DOCS to the Obsidian vault repo.
- Documented the vault layout in CONTEXT: spec/decision notes at root; per-operator
  session logs under `History/Josh/` and `History/Michael/` as `session-<n>.md`.
- Added the rule: **the vault always commits straight to `main` and pushes** (knowledge
  base kept fresh); the project keeps its branch + PR flow.
- Committed on branch `chore/loop-obsidian-session-history` (commit `a07120c`), pushed.
- Removed the now-dead `stitch` MCP server from `.mcp.json` (local, gitignored, not shared).

## Claude Design integration — test outcome (PASS, with caveat)
Pulled Michael's `130586b "migration from stitch to claude design system"` into main
(fast-forward). Verified:
- `docs/design/claude-design/` bundle present: 2 foundations + 7 screen cards.
- **Upload contract holds** — every HTML file's line 1 is a valid `@dsCard` marker,
  no BOM, no leading blank line.
- `design-to-code` agent + root `DESIGN.md` rewired from Stitch to Claude Design /
  `DesignSync` / `/design-sync`.
- `DesignSync` tool is wired; `get_project` returned the **documented** auth gate —
  live push needs an interactive `/login` (a `CLAUDE_CODE_OAUTH_TOKEN` session can't
  be granted design scopes). Not a defect.

## Resolved this session
- **Vault git is now set up** — `hb-vault` connected to `https://github.com/michaeljvr11/hb-vault.git`
  (Michael's repo). This session log was committed straight to `main` and pushed.
- **Dead `stitch` MCP server removed** from `.mcp.json` (local/gitignored).
- **Missing `resend` dep** — `apps/api` declares `resend@^4.8.0` but `node_modules` was stale,
  failing the API test gate. Ran `npm install`; suite green (22 tests). Pre-existing, not from
  this change.

## Follow-ups
- **Open the project PR** — `gh` is not installed and no API token is available, so the merge
  request could not be auto-created. Branch is pushed; create it at:
  https://github.com/michaeljvr11/hb-mono-repo/pull/new/chore/loop-obsidian-session-history
- Consider installing `gh` so the `/ship-card` DELIVER step can open PRs unattended.
