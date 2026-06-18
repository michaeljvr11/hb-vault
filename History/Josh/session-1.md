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
- Committed on branch `chore/loop-obsidian-session-history` (commit `5c973a6`).

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

## Follow-ups
- **Vault is not a git repo on this machine** — `hb-vault` has no `.git`, so this
  session log could not be committed/pushed to the vault. Needs `git init` + remote
  (or the agreed sharing mechanism) before the loop's COMMIT DOCS step can run.
- `.mcp.json` still defines the `stitch` MCP server — now dead config post-migration;
  Michael should decide whether to remove it (it carries a machine-specific path + key).
- The loop change on `chore/loop-obsidian-session-history` is unpushed — open a PR when ready.
