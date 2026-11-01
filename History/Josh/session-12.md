---
operator: Josh
date: 2026-06-18
session: 12
tags: [dev-environment, mcp, onboarding, retroactive]
---

# Session 12 — Local dev environment + MCP connectivity

> Logged retroactively on 2026-11-01 from chat history. This was the first working session
> (before session 1), so it is numbered out of order. It was picked up again on 2026-08-22.

## What we worked on
Getting the mono-repo running on Josh's machine and the agent tooling connected, before any card work.

## What happened
- **Postgres:** Michael runs Rancher, Josh runs Podman. Cleared whatever was holding port 5432 and
  recreated the project database container so both machines behave the same.
- **Preview:** ran the app in the Claude app preview via `.claude/launch.json`.
- **Git identity:** checked the GitHub credentials on the machine and switched everything to the
  personal account (`joshdasilva5501`), not the work (`@entelect`) one.
- **MCP servers:** reviewed every configured connection. Fixed Obsidian by installing the Local REST
  API plugin and adding its bearer token. Trello took several attempts: a Power-Up only shows your
  own workspace, and Josh is a guest on Michael's board. The cause turned out to be a mis-pasted token,
  not a paid-plan limit. A test card was created to prove it.

## Revisit — 2026-08-22
The preview could no longer bind 5432. The cause was a native PostgreSQL 18 Windows service that had
re-enabled itself and grabbed the port, so the Podman `hb-postgres` container had exited.
Also decided that the database does not belong in the preview launch config: `npm run db:up` detaches
straight away, so the preview can't monitor it. Removed the `db` entry.

## Key decisions
- One project-owned Postgres container on 5432 on every machine. Nothing else gets to hold that port.
- Commits and GitHub auth use the personal identity only.

## Follow-ups
- Keep the native PG18 service disabled, or it will take 5432 again after updates.
