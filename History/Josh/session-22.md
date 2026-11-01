---
operator: Josh
date: 2026-06-18
session: 22
tags: [ai-factory, design-sync, claude-design, multi-dev, retroactive]
---

# Session 22 — Design-sync config per device

> Logged retroactively on 2026-11-01 from CLI history and PR #6.

## What we worked on
`/design-sync` hit Michael's project id, and Josh wanted that to be known information "so we don't
spiral into these same patterns". He asked for the professional approach, not a shortcut, so that
multiple devs can collaborate without friction.
[PR #6](https://github.com/michaeljvr11/hb-mono-repo/pull/6), merged 2026-06-18.

## Problem
claude.ai/design projects are owned **per account**. The `projectId` committed in
`.design-sync/config.json` (one dev's UUID) returns HTTP 404 on the other dev's device. That looks like
a deleted project and tempts an agent to repoint or recreate it, which just moves the 404 to whoever
syncs next.

## What changed
- `.design-sync/config.json` is now git-ignored and treated like `.env`. It holds each device's own
  resolved `projectId`.
- A committed `.design-sync/config.example.json` template holds `projectName` + `localDir`, with no UUID.
- `.design-sync/NOTES.md` documents the one-time resolve-by-name bootstrap and the "404 is expected" rule.
- The project UUID was removed from the bundle `README.md`, the `DESIGN.md` frontmatter and
  `docs/AI_FACTORY.md`, which now name the design system instead.

## Key decisions
- Each dev resolves their own same-named project once (`DesignSync list_projects`). No project UUID is
  tracked in git.
