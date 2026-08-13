# Session 29 — EDR-1: Earnings date-range picker

**Date:** 2026-08-13 · **Card:** [EDR-1](https://trello.com/c/2bWAyVnD) · **Branch:** `feat/EDR-1-earnings-date-range-picker` · **Status:** PR about to open

- Shipped: Added "All time" preset + custom date-range picker to vendor/admin earnings screens. Extracted duplicate selector logic into one shared component (`apps/web/src/app/shared/components/earnings-range-selector/`). API: epoch sentinel `ALL_TIME_START`, `@IsNotFutureDate()` guard, index migration on `orders.createdAt` and `order_items.vendorId`. Client-side pagination of settlement periods (page size 10, defaults to last page). EARNINGS_WINDOWS contract gains `'all'`.
- Decisions: Folded index migration (question 3) into this card despite earlier "separate card" note — pragmatic call given load-bearing nature for all-time queries. UTC/local cap mismatch fixed client-side: adapter builds cap from UTC parts fed to local Date constructor, preserving local-parts formatting (toISOString would drift day in SAST). Month tab label now from wall clock, never from report `from`, to prevent "January 1970" label leak when sentinel is selected.
- Tests: API 631/631 green (earnings specs 45/45), web 801/801 green (selector states, datepicker, pagination, UTC/local boundary), lint:api clean, build clean.
- Follow-ups: Composite `(status, createdAt)` index would better serve the actual earnings predicate than the single-column shipped here; flag with the existing "push aggregation into SQL" follow-up.
