# Session 45 — ESMDKpTW: SEO prerender fixes (two cards)

**Date:** 2026-09-21 · **Cards:** E0knLhvG + ESMDKpTW · **Branch:** `feat/ESMDKpTW-seo-prerender-fixes` · **Status:** open

- **Shipped:** E0knLhvG — `CategoryNav` now skips the live category fetch on build-time prerender by injecting `REQUEST` (present only at request-time) and guarding `store.load()` — build succeeds against unreachable API. ESMDKpTW — `/robots.txt` and `/sitemap.xml` endpoints at request-time (not build-time), origin derived from incoming request, sitemap cached 15-min in-memory with 10-second budget, canonical URL helper added to PDP/vendor-profile, all routes/pages included.

- **Decisions:** E0knLhvG — reused existing `inject(REQUEST, {optional})` pattern rather than new mechanism. ESMDKpTW — dynamic sitemap/robots generation (no baked hostname), categories via `/discover?categoryId=`, canonical URLs self-referential only, `PUBLIC_API_URL` reconciliation deferred until live deployment.

- **Tests:** web 91 files / 1348 tests passed; build clean with unreachable API; E0knLhvG code review SHIP, ESMDKpTW first pass FIX-FIRST (4 findings fixed, re-review SHIP, non-blocking scalability note for 12k+ products).

- **Follow-ups:** ESMDKpTW deploy-time checks (verify absence of hairpinned cache fetches, verify 10-second budget behavior); E0knLhvG local end-to-end with real Meilisearch data (blocked by pre-existing local-dev gap).
