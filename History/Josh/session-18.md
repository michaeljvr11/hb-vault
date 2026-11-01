---
operator: Josh
date: 2026-09-14
session: 18
tags: [ship-card, prod-readiness, ssr, retroactive]
---

# Session 18 — PROD-1: runtime API base URL (KYwTmZyB)

> Logged retroactively on 2026-11-01 from chat history and PR #84.

## What we worked on
Card PROD-1 "environment-portable web image", branch `feat/KYwTmZyB-runtime-api-base-url`.
[PR #84](https://github.com/michaeljvr11/hb-mono-repo/pull/84), merged.

## What changed
| File | Why |
|---|---|
| `environment.ts` | `apiBaseUrl` is now `${location.origin}/api` at runtime, so no hostname is baked into the bundle |
| `app.config.server.ts` | The transfer-cache origin map is built per request from `REQUEST.url`, with a module-init assertion |
| `Dockerfile` | A `PRERENDER_API_BASE_URL` build arg feeds the build-time prerender |
| `api-base-url.guardrail.spec.ts` | New. Pins the invariant so it can't silently regress |
| `apps/web/CLAUDE.md` | Documents the contract |

## Key decisions
- **The card's option 1 (a bare relative `/api`) doesn't work as written.** `new URL(apiBaseUrl).origin`
  throws on it, and the transfer cache silently dies, which is the double-fetch the code was guarding
  against. The browser needs an absolute origin derived at runtime instead.
- At build-time prerender there is neither `location` nor a real request. `/api/...` was resolving
  against Angular's prerender server, which is why the build-arg is needed.
- Built the base branch cleanly *before* the change, to prove the prerender break came from this change.
- The leftover "outstanding changes" were **not** discount work. They were LAN-IP dev overrides, and
  committing them would have broken other devs and contradicted this card, so they stayed in a stash.

## Not verified
- AC1, "one image serves two public hostnames", needs a second hostname, which won't exist until PROD-3.
  This was stated in the PR rather than ticked.

## Follow-ups
- The ten prerendered marketing/legal routes fetch `/api/categories` from the live dev box at build
  time, so the artefact depends on whatever CI can reach. A follow-up card was flagged in the PR.
- PROD-2 is still gated on the secrets-store and backup-destination decisions (see [[session-17]]).
