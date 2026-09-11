# Session 42 — 83G6w4yZ: SSR API Routing — Internal Hairpin & Throttle Identity

**Date:** 2026-09-11 · **Card:** 83G6w4yZ · **Branch:** `feat/83G6w4yZ-ssr-internal-api-routing` · **Status:** open

- Shipped: SSR API calls now route internally (`http://api:3000/api`) instead of hairpinning through the public URL; `x-forwarded-for` forwarded verbatim so `ThrottlerModule` no longer collapses the entire catalogue into one 120 req/min bucket. Functional interceptor + injectable token, boot-time config validation, transfer-cache origin-map fix (caught in code review — avoids a silent double-fetch on every SSR page).
- Decisions: Verbatim XFF (not appended) is load-bearing — appending makes Express resolve `req.ip` to the proxy and re-collapses the bucket. Interceptor is *registered* in `app.config.ts` (Angular has no public API for adding a functional interceptor from a separate config); its server-only behaviour rides on `INTERNAL_API_BASE_URL`, which only `app.config.server.ts` provides. `HTTP_TRANSFER_CACHE_ORIGIN_MAP` maps origins only, so the internal URL must keep the `/api` suffix. Prefix match anchored so `/api-docs` isn't dragged along. Malformed config aborts at boot, before port bind.
- Tests: 1291/1291 web (incl. 14 new interceptor specs + a mutation check), `npm run build` clean. API suite not run — this card touches no `apps/api` or `libs/shared` code; CI re-runs it as the PR gate.
- Follow-ups: Two post-deploy acceptance criteria remain (Caddy access log clean, load test zero 429s); all local verifications complete.
