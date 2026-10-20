---
operator: Michael
date: 2026-10-20
session: 57
tags: [ai-factory, ship-batch, contrast, fonts, delivery]
---

# Session 57 — SVO-14, self-hosted fonts, OPT-6: delivery-pipeline hygiene batch

**Date:** 2026-10-20 · **Cards:** [SVO-14](https://trello.com/c/gxMLsjiq) · [fonts](https://trello.com/c/qxdF3zoY) · [OPT-6](https://trello.com/c/qamP5k8h) · **Branch:** `feat/gxMLsjiq-contrast-gate-fonts-slim-images` · **Status:** open, [PR #124](https://github.com/michaeljvr11/hb-mono-repo/pull/124)

- Shipped: CI contrast gate on design tokens (48 pairs, `check:contrast`). Inter and Material Symbols self-hosted, Google Fonts dropped from the CSP. Per-app prod-deps images and keep-last-5 image pruning on deploy (OPT-6). Code review SHIP after hardening.
- Decisions: tier 1 only, nightly browser contrast job deferred. `inlineCritical` stays off (Beasties onload breaks under nonce-only CSP). Font packages as devDependencies. Mode: wave 1 parallel (SVO-14 + fonts, disjoint files), wave 2 OPT-6 after fonts; bundled at owner request.
- Tests: web 109 files / 1715 pass; contrast gate 12/12 node:test cases; prune script PASS locally.
- Follow-ups: tier-2 nightly contrast job; real-box check of prune and slim images after deploy; consider a `npm ls --omit=dev` assertion in prod-deps.
