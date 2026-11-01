---
operator: Josh
date: 2026-06-18
session: 21
tags: [ai-factory, hooks, vault, retroactive]
---

# Session 21 — Vault auto-pull on session start

> Logged retroactively on 2026-11-01 from chat history (desktop "Project preview launch").

## What we worked on
Launched the project in preview, then asked for the Obsidian vault to be pulled automatically.
Josh chose "when a Claude session starts" over a timed job, so this became a `SessionStart` hook,
not a scheduled task.

## What changed
- `.claude/hooks/pull-vault.js`, registered under `hooks.SessionStart` in `.claude/settings.json`
  with a 30s timeout. It fast-forwards `hb-vault`, resolved as the sibling of the mono-repo and
  confirmed by its `hb-vault` origin remote.
- The hook is defensive because it runs unattended:
  - it never blocks the session (always exits 0)
  - it skips and warns if any note file is dirty or local `main` has diverged
  - it only stashes Obsidian's disposable `.obsidian/` churn, using `--ff-only`
  - it logs `vault_pull` events to the factory log
- A plain `git pull` would have failed every session, because Obsidian keeps dirtying
  `.obsidian/workspace.json`. This was proven by breaking it first.

## Status
- The commit (`f437778`) sits **only on the local branch `chore/vault-session-start-pull`**. It was
  never pushed or merged, so the hook isn't live. This is why the vault drifted 37 commits behind on
  this machine by November.

## Follow-ups
- Push that branch and open a PR, or fold the hook into the factory session-logging fix.
