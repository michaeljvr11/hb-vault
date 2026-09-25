# Session 47 — bNZweqNF, S3IEP59u, XWbllttP, 1IOweJ0t, I4RPWquJ: Storefront prelaunch batch

**Date:** 2026-09-25 · **Cards:** bNZweqNF, S3IEP59u, XWbllttP, 1IOweJ0t, I4RPWquJ · **Branch:** feat/bNZweqNF-storefront-prelaunch-batch · **Status:** PR pending

- **Shipped:** Currency switcher display-only (browse converts via exact BigInt half-up rounding; cart/checkout settle in listed currency with on-cart note), peg rate now data (`platform_settings.nadPerZar`), newsletter capture with throttling + admin UI, vendor `description` on public DTO, rating distribution bars at any review count, back-office loading states (row skeletons + `app-state-message`).
- **Decisions:** Display-only currency (settlement unchanged); peg as `platform_settings` data; default display-currency = latest user address country else NAD; vendor description inherits privacy boundary (public safe); two-wave parallel→sequential execution avoiding Fable offload; skip review count guard on PDP bars.
- **Tests:** API 81 suites / 1133 tests green; Web 98 files / 1491 tests green; lint clean; build green; review round 1 found admin peg input crash, null→500 regression, half-up rounding edge cases — all fixed in-branch; confirming pass SHIP, then W1/W2 (bad-rate guard, sign-out race) fixed in-branch.
- **Follow-ups:** class-validator `maxDecimalPlaces` 500s on exponent-form numbers (~7 DTOs incl. public search minPrice/maxPrice) — separate card; POPIA copy before real newsletter sends; admin-table SCSS partial dedup; pre-existing SCSS budget warnings.
