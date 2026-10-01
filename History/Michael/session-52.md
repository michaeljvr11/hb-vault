---
operator: Michael
date: 2026-10-01
session: 52
tags: [ai-factory, ship-batch, performance, deployment]
---

# Session 52 — OPT-5 + PROD-3 batch

**Date:** 2026-10-01 · **Cards:** [OPT-5](https://trello.com/c/sqEp2xXj) (API image-processing hardening) · [PROD-3](https://trello.com/c/MM6LVEVi) (deploy pipeline) · **Branch:** `feat/MM6LVEVi-deploy-pipeline-and-api-memory` · **Status:** PR to be opened

## Shipped

**OPT-5:** jemalloc preloaded in the Dockerfile runtime stage, sharp config (cache/concurrency) at bootstrap, FIFO semaphore gate on image processing (IMAGE_PROCESS_MAX_CONCURRENT default 2), SearchIndexerService.runFullReindex now paged by id cursor (500 rows/page) with bounded memory. Verified locally (jemalloc in /proc/self/maps); full test suite 1428 pass. Not yet verified: docker stats peak measurement, in-container /proc/1/maps jemalloc check — awaits deployed box.

**PROD-3:** deploy.yml reusable workflow (workflow_call: environment, sha); ci.yml calls it for dev on push to main; new deploy-prod.yml triggers on push to prod branch (requires sha on main + successful dev Deployment). Images tagged per main SHA, promotion is `git push origin <sha>:prod` (fast-forward, no merge commit). Per-environment concurrency, no hostname literal (gates build uses prerender-placeholder.invalid). Owner must add SITE_URL to GitHub dev Environment before merge or dev deploy fails by design. New runbooks: prod-server.md, dev-server.md updated. Code review fixes (FIX-FIRST, done): (1) SSH_*/SITE_AUTH_* secrets were repo-level, would leak prod-to-dev via secrets:inherit — moved to dev Environment in PROD-4; (2) prod Environment needs deployment-branch policy limited to prod branch; (3) verify job reads newest non-inactive deployment. Not verifiable until prod exists: Deployments API filtering, vars.SITE_URL resolving, secrets:inherit, approval gating, concurrent dev+prod. Gotcha: prod-fence hook matches substring 'prod' in git push, so feature branches named '...-prod-...' are blocked (branch renamed to ...-deploy-pipeline-...).

## Decisions

- Scope amended 2026-10-01 (owner): PROD-3 ships code/docs that build without prod box; activation is new card PROD-4 (prod box setup + first deployment).
- OPT-5 follow-ups: (a) prune race window — product created mid-reindex can be pruned until next update/3am if uuid sorts below cursor; (b) semaphore bounds libvips but not queued multer buffers (queue depth unbounded).

## Tests

- API 92 suites / 1428 tests pass, lint clean, full build clean.
- New OPT-5 specs: semaphore.spec.ts, sharp-tuning.spec.ts, image-processor.concurrency.spec.ts, plus paging/prune cases in search-indexer.service.spec.ts.
- PROD-3: YAML parse + zero-hostname grep verified locally; actionlint not installed.

## Follow-ups

- OPT-5 docker stats verification (not done; awaits deployed image).
- PROD-4: move SSH_*/SITE_AUTH_* secrets from repo-level to dev Environment, delete repo copies.
- Optional: document SHARP_* / IMAGE_PROCESS_MAX_CONCURRENT env vars in docker-compose.prod.yml, .env.example, runbook.
