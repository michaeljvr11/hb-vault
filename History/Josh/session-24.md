---
operator: Josh
date: 2026-07-03
session: 24
tags: [design-sync, claude-design, tooling, retroactive]
---

# Session 24 — Design-sync login attempt

> Logged retroactively on 2026-11-01 from CLI and desktop history. Short session, no code.

## What happened
- Ran `/design-sync` from a desktop session. It failed because DesignSync needs an interactive terminal
  login (`/design-login`), which isn't available in a desktop or headless session.
- Retried from the CLI with `/design-login` then `/design-sync`, but from the home folder instead of the
  repo, so the project context had to be pointed back at `hb-mono-repo`.

## Key decisions
- The local bundle at `docs/design/claude-design/` is the source of truth. The per-account claude.ai/design
  projects are only render targets synced from it, so nothing is lost when a sync can't run.
- Sync options: run `/design-sync` from an interactive terminal in the repo (it resolves the project by
  name, see [[session-22]]), or use Claude Design's "Send to Claude Code Web".
