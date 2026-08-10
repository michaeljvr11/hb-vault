# Session 26 — VE-5: Vendor own-earnings report

**Date:** 2026-08-10 · **Card:** VE-5 (#74, `WrPOgQDa`) · **Branch:** `feat/WrPOgQDa-vendor-own-earnings` · **Status:** PR not yet opened

- **Shipped:** Vendor-scoped earnings report endpoint `GET /vendors/me/earnings` + portal UI. Vendor sees accrued balance, settlement preview (current bi-weekly period + closed history), and per-currency net totals. Mirrors VE-4's admin report structure but ownership server-resolved from auth, no vendorId query param.
- **Decisions:** `periodEnd` rendered as last-inclusive-day (not raw exclusive boundary) to prevent visual overlap with adjacent periods; eligible-lines-only summary logic (accrued + settlementPreview) deliberately mirrors not imports VE-4's private methods to keep module boundaries clean.
- **Tests:** api 586/586 (42 suites, +14), web 725/725; lint/build clean.
- **Follow-ups:** 5th copy of `CURRENCY_SYMBOLS`/`formatMoney`, accessor sprawl, verbose HttpParams builder — candidate follow-up, not fixed.
