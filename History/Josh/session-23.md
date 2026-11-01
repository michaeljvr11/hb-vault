---
operator: Josh
date: 2026-06-19
session: 23
tags: [ai-factory, tooling, git, merge-conflicts, evidence, retroactive]
---

# Session 23 — Merge conflicts + union merge driver for evidence files

> Logged retroactively on 2026-11-01 from CLI history and PR #13.

## What we worked on
An open PR's branch was behind, so we checked it out, resolved the conflicts and updated it. The
conflicts came back on `factory-log.jsonl` and the AI evidence files, so Josh asked for a strategy that
stops them happening at all.
[PR #13](https://github.com/michaeljvr11/hb-mono-repo/pull/13), merged 2026-06-19.

## Root cause
Every parallel branch appends to `.claude/factory-log.jsonl` and regenerates
`docs/ai-evidence/REPORT.md`. `.gitattributes` pointed at a custom `merge=evidence-regen` driver that
was never installed locally or on GitHub, so git fell back to a 3-way text merge every time.

## What changed
- `.gitattributes`: uses the built-in `merge=union` on all four evidence files. It takes lines from
  both sides with no conflict markers, works in GitHub's web merge, and needs no local setup.
- `.claude/hooks/post-merge.sh`: a new git hook that runs `npm run evidence` after any local merge or
  pull, so the report reflects both branches.
- `.claude/hooks/install-merge-drivers.sh`: simplified to just install `post-merge.sh` into `.git/hooks/`.
- `CLAUDE.md`: running `sh .claude/hooks/install-merge-drivers.sh` is now a required one-time step after
  every fresh clone.

## Follow-ups
- Both machines need to run `git pull` and then `sh .claude/hooks/install-merge-drivers.sh` once.
