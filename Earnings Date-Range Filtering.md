# Earnings Date-Range Filtering

Follow-up to [[Vendor Earnings & Commission]] (VE-1 → VE-5, all shipped). Adds an
**"All time"** preset and a **custom start/end date picker** to the earnings range
selector. Requested by Michael, 2026-08-11.

## Problem

Both earnings screens expose exactly three hard-coded range presets — "Last week",
"Last 2 weeks", and the current calendar month. There is no way to see the full
history, and no way to ask a question like "what did we earn in March?" without
editing the URL by hand.

The important finding: **this is a UI gap, not an API gap.** The API has accepted
arbitrary `from`/`to` since VE-4. The web app simply never renders a control that
sends them.

## Current implementation (verified)

### Where the screens live

There is **no separate admin app**. Both portals are role-gated route trees inside
the single `apps/web` Angular app, so this ships to **both** — two sibling
components, not one shared one:

- Vendor: `apps/web/src/app/app.routes.ts:94` → `/vendor/earnings` →
  `apps/web/src/app/features/vendor/pages/vendor-earnings/vendor-earnings.ts`
- Admin: `apps/web/src/app/app.routes.ts:116` → `/admin/earnings` →
  `apps/web/src/app/features/admin/pages/admin-earnings/admin-earnings.ts`

The two components **duplicate the selector logic verbatim**: an identical
`WindowTab` interface, an identical `windowTabs` computed with the same three
entries, an identical `selectedWindow` signal, and an identical `setWindow()`
(`admin-earnings.ts:12-15,33,55-59,80-84` and `vendor-earnings.ts:12-15,33,49-53,74-78`).
The markup is likewise duplicated (`admin-earnings.html:16-30`).

### The contract today

`libs/shared/src/contracts/earnings.ts`:

- **`EARNINGS_WINDOWS = ['1w', '2w', '1m']`** (line 58) with
  `EarningsWindow = (typeof EARNINGS_WINDOWS)[number]` (line 59). The doc comment
  is explicit that this array is *the* source of truth and that runtime validators
  must derive from it — which they do, so **adding a value here automatically
  widens API validation**.
- **`AdminEarningsQuery`** (line 68) — `window?`, `from?`, `to?`, `vendorId?`
- **`VendorEarningsQuery`** (line 140) — `window?`, `from?`, `to?` (deliberately no
  `vendorId`; vendor scope is always server-resolved — ownership boundary)

`from`/`to` are documented as "ISO date (yyyy-mm-dd), inclusive. Must be supplied
together."

### The API already does custom ranges

`apps/api/src/common/utils/earnings-window.utils.ts` — `resolveEarningsWindow()`:

- lines 30-43: explicit `from`/`to` **already win** over `window`, are already
  expanded to `startOfDayUTC` / `endOfDayUTC`, and already throw
  `BadRequestException` when supplied alone (line 32) or when `from > to` (line 39).
- line 45: defaults to `'1m'` when nothing is supplied.

Both DTOs already validate the fields — `AdminEarningsQueryDto`
(`apps/api/src/admin/dto/admin-earnings-query.dto.ts:10-16`) and
`VendorEarningsQueryDto` (`apps/api/src/vendors/dto/vendor-earnings-query.dto.ts:15-22`),
each `@IsOptional() @IsDateString()`, each `@IsIn(EARNINGS_WINDOWS)` for the preset.

The web API clients already forward them too:
`apps/web/src/app/core/api/earnings.service.ts:22-27` sets `from`/`to` on the
`HttpParams` whenever present. **Nothing calls it with them.**

Endpoints: `GET /api/admin/earnings`
(`apps/api/src/admin/admin.controller.ts:68-71`) and `GET /api/vendors/me/earnings`
(`apps/api/src/vendors/vendor-earnings-report.service.ts:62`).

### How earnings are aggregated

`apps/api/src/earnings/vendor-earnings.service.ts` — `getEarnings()` (line 111) and
`getEarningsByVendor()` (line 241). Both:

1. Build a query bounded by **`o.createdAt BETWEEN :from AND :to`** (lines 121, 251),
   filtered to `oi.vendorId IS NOT NULL` and `o.status = delivered`.
2. Call **`query.getMany()`** (lines 132, 258) — every matching `order_item` plus its
   joined `order` is hydrated into memory.
3. Aggregate **in JavaScript**, per line, in integer cents (`splitGrossNetCents`,
   line 440).

`AdminEarningsService.getPlatformListingGmv()`
(`apps/api/src/admin/admin-earnings.service.ts:204-234`) follows the same
`getMany()`-then-loop-in-JS shape.

**This matters for "All time":** the date range is the only thing bounding the
result set, so removing it means an unbounded scan and full hydration of the vendor
order-item history. See "Open questions".

### Existing test coverage (good news)

Money logic here is already well covered and the new range only has to slot into it:

- `apps/api/src/common/utils/earnings-window.utils.spec.ts` — 15 cases covering
  every preset, month/year rollover, leap-February, `from`/`to` precedence, and all
  three throw paths.
- `apps/api/src/earnings/vendor-earnings.service.spec.ts`,
  `apps/api/src/admin/admin-earnings.service.spec.ts`,
  `apps/api/src/vendors/vendor-earnings-report.service.spec.ts`
- Web: `admin-earnings.spec.ts`, `vendor-earnings.spec.ts`

## Business rules it must honour

From [[Vendor Earnings & Commission]] — a wider range must not restate any of these:

1. **Eligibility is unchanged by the range.** A line counts only when the order is
   `delivered` **and** its 48-hour damage-claim window has elapsed
   (`DAMAGE_CLAIM_WINDOW_HOURS`). "All time" widens *which lines are in scope*; it
   never makes an ineligible line eligible.
2. **Both reports are eligible-lines-only.** `pendingClaimWindow` is excluded from
   `summary` / per-vendor rows. Still true at any range.
3. **Settlement periods stay anchored to `SETTLEMENT_ANCHOR_DATE`** and bucket on the
   eligibility instant (`deliveredAt + 48h`), not `deliveredAt`. A custom range must
   not re-bucket anything — it only filters which periods appear.
4. **ZAR and NAD are never summed.** Per-currency `CurrencyTotalDto` entries only.
5. **Rounding is per-line**, commission half-up to 2dp with net derived by
   subtraction, so `commission + net === gross` exactly. A range change must not
   introduce a re-aggregation path that rounds twice. ([[Money & Currency Rules]])
6. **`heldForVendorsByCurrency` is a "right now" snapshot, not window-scoped** —
   already documented on `AdminEarningsReportDto` (contract lines 121-129). It will
   read identically under "All time" as under "Last week". Not a bug; make sure the
   UI copy doesn't start implying otherwise.
7. **Payout cadence is bi-weekly and is not affected.** This feature is
   visibility/reporting only — no money moves, no payout is executed. The vault is
   emphatic on this and it stays true here.

## Scope

**Ships to both the vendor portal and the admin portal** (both live in `apps/web`).

1. Add an **"All time"** option to the range selector on both screens.
2. Add a **custom date range picker** (start + end) on both screens, wired to the
   already-supported `from`/`to` query params.
3. Extract the now-four-preset selector + custom range control into **one shared
   component** rather than triple-duplicating it. The current copy-paste between the
   two components is exactly what makes this change twice the work.

## `@hb/shared` contract impact

Deliberately minimal — **the generic range support already exists; only the preset
list grows.**

- **`EARNINGS_WINDOWS`** (`libs/shared/src/contracts/earnings.ts:58`) gains `'all'`:
  `['1w', '2w', '1m', 'all']`. Because both DTOs derive `@IsIn(EARNINGS_WINDOWS)`
  from this array, API validation widens with no DTO edit needed — this is the
  documented intent of that array.
- **`AdminEarningsQuery` / `VendorEarningsQuery` are unchanged.** `from`/`to` already
  exist, are already validated, and are already resolved. The custom range picker
  needs **zero** contract change.
- **Existing presets stay** as shorthand on top of the generic range support. They
  are not reimplemented as client-computed `from`/`to` — the server keeps owning
  "what does `1m` mean", which is what keeps the calendar-month semantics honest.
- `resolveEarningsWindow()` gains an `'all'` branch. No response DTO shape changes:
  `AdminEarningsReportDto.from`/`.to` and `VendorEarningsReportDto.from`/`.to` still
  echo the resolved range back.

**No migration required** unless the indexing question below is answered "add the
index" — in which case a TypeORM migration is mandatory per the non-negotiables.

## Out of scope

- Executing payouts, settlement batches, bank transfers — unchanged from
  [[Vendor Earnings & Commission]], still out of scope entirely.
- CSV / PDF export, scheduled report delivery.
- Any new preset beyond "All time" (no "last quarter", "year to date", "last 7 days").
- Changing what `1w` / `2w` / `1m` mean.
- The `/admin/analytics` and vendor-dashboard range selectors — `AdminAnalyticsQuery`
  is a **separate** query shape with deliberately different (30-day-rolling) default
  semantics. `resolveEarningsWindow` was written as a sibling to
  `resolveAnalyticsRange`, not a reuse of it. Do not unify them under this card.
- Timezone selection. Everything stays UTC-pinned, as the existing utils are.

## Open questions

1. **What is `from` for "All time"?** Two options, and this affects what the UI
   renders in the `from`–`to` sub-header (`admin-earnings.html:41`):
   - **(a) A sentinel** (e.g. epoch / a named `ALL_TIME_START` constant) — cheap, no
     extra query, but the header would read "1 Jan 1970" unless the UI special-cases
     the "All time" label.
   - **(b) The earliest `order.createdAt` in scope** — honest dates, but costs an
     extra query and differs per vendor scope.
   *Recommendation: (a) with the UI rendering "All time" instead of the raw date.*
   **Needs a human decision.**

2. **Performance of an unbounded scan.** With the date range removed, `getMany()` on
   `order_items` joined to `orders` hydrates the entire eligible history into Node
   and aggregates it in JS. At current data volumes this is fine; it does not scale.
   Is "make it work now, revisit when volume demands" acceptable, or should this card
   also push the aggregation into SQL? *Recommendation: ship it, measure, and raise a
   separate card if a real dataset shows a problem — pushing the aggregation into SQL
   is a rewrite of well-tested money code and does not belong in a UI card.*

3. **`orders.createdAt` is not indexed.** Confirmed: the only order-table index added
   anywhere in `apps/api/src/database/migrations/` is `IDX_orders_deliveredAt`
   (`1784505600000-OrderDeliveredAt.ts:24`). There is no index on `orders.createdAt`
   and none on `order_items.vendorId`, both of which every earnings query filters on.
   This is a **pre-existing** gap, not one this feature creates — but "All time" and
   wide custom ranges are what will first make it hurt. Should an index migration be
   folded into this card or tracked separately? *Recommendation: separate card — it
   is a schema change with its own migration and its own review surface.*

4. **How many settlement-period rows should "All time" render?** The vendor screen
   lists every closed bi-weekly period in range. Over years that is an unbounded
   table. Cap it, paginate it, or accept it? **Needs a human decision.**

5. **Should the custom range allow a future `to` date?** `resolveEarningsWindow`
   currently does **not** reject future dates — it accepts them and simply returns
   everything up to that point. Blocking future dates in the picker is the sensible
   UX; blocking them server-side is a behaviour change to an existing validated
   endpoint. *Recommendation: cap the picker at today (client) and add the server-side
   guard too, since a future `to` can only ever be a mistake.*

## Adjacent observation (not a blocker)

`AdminEarningsService.getReport()` calls `resolveEarningsWindow(query)` without
threading a `now` (`apps/api/src/admin/admin-earnings.service.ts:79`), then
`getEarningsByVendor(...)` reads the clock again independently — two clock reads per
request. `VendorEarningsReportService.getMyEarnings()` threads a single `now`
correctly (`vendor-earnings-report.service.ts:70-77`). Harmless today; worth
tidying if "All time" makes `to = now` load-bearing.

## Implementation Notes

**Date:** 2026-08-13 · **Card:** [EDR-1](https://trello.com/c/2bWAyVnD) · **Branch:** `feat/EDR-1-earnings-date-range-picker` · **PR:** About to open

### Resolution of the five open questions

1. **"All time" `from`** — Resolved as **option (a), sentinel**. `ALL_TIME_START` constant exported from `apps/api/src/common/utils/earnings-window.utils.ts`; `resolveEarningsWindow()` returns a FRESH `Date` per call so a caller mutating the returned `from` can't corrupt later calls. The UI renders the label "All time" instead of the raw date, so the sentinel never leaks to the screen. This was the recommendation and it shipped that way.

2. **Unbounded-scan performance** — Shipped as-is, aggregation left in JavaScript. The recommendation held: pushing aggregation into SQL is a rewrite of well-tested money code and does not belong in a UI card. Volume is acceptable at present; a follow-up card will track it if future data demands.

3. **`orders.createdAt` not indexed** — **Resolved AGAINST the note's "separate card" recommendation.** Michael folded it into this card: migration `1786665600000-EarningsRangeIndexes.ts` adds `IDX_orders_createdAt` and `IDX_order_items_vendorId`. TypeORM runs migrations in a transaction so `CONCURRENTLY` is unavailable; it takes a SHARE lock (reads continue, writes block for the build). This is a pragmatic call: the index is load-bearing for all-time queries, and splitting it would mean a second PR cycle.

4. **Settlement-period rows under "All time"** — Paginated client-side at page size 10. The client receives `VendorEarningsReportDto.settlementPreview` as an unchanged array; pagination is applied at render time. The page defaults to the LAST page on load so the always-present `open` current period is visible without requiring the user to paginate.

5. **Future `to` date** — **Capped in both the picker AND guarded server-side.** Client cap: the picker's max date is the local today. Server guard: both `AdminEarningsQueryDto` and `VendorEarningsQueryDto` validate with `@IsNotFutureDate()` on the `to` field. This is an intentional behaviour change to two existing endpoints (they previously accepted future dates silently); it is not a bug — it blocks a mistaken input client-side and redundantly at the boundary.

### Key implementation decisions

- **Contract change: one line.** `EARNINGS_WINDOWS` array gains `'all'`; because both DTOs derive `@IsIn(EARNINGS_WINDOWS)` from it, API validation widened with no DTO edit needed — the array's documented intent, now proven by test coverage.

- **One shared selector component.** Extracted `apps/web/src/app/shared/components/earnings-range-selector/` consumed by both vendor and admin pages. Deleted the duplicated `WindowTab`/`windowTabs`/`setWindow`/`monthLabel` blocks from both original components; markup deduplication avoids future divergence.

- **Two bugs caught in review — both worth documenting as traps:**
  (a) **Month tab label leak:** The tab was initially labelled from the loaded report's resolved `from`, so selecting "All time" relabelled it **"January 1970"** — the sentinel leaking into UI text. Fixed by labelling from the wall clock: the month tab means "the current calendar month" and is never a function of the report's range.
  (b) **Date picker UTC/local mismatch:** The picker capped at LOCAL today while the server guard rejects on UTC today. This meant a SAST (UTC+2) user between 00:00 and 02:00 could select a date the server then 400d. Fixed by building the cap from UTC parts fed into the local `Date` constructor — the adapter compares local parts, so the cap stays in its own coordinate system while agreeing with the server. **Date FORMATTING stays local-parts** (`toISOString()` would shift the day back in SAST); only the cap is UTC-derived. This mixed-coordinate detail is the single most confusing part of the component and must survive in the code comments for the next maintainer.

- **Angular ZONELESS:** This app runs without `zone.js` polyfill, so `fakeAsync` and `tick()` are unavailable in specs. Debounced selector changes await a real delay instead. Worth documenting because the next timing test will hit this.

- **Accounting unchanged, as required.** The 48-hour damage-claim gate, eligible-lines-only reporting, `SETTLEMENT_ANCHOR_DATE` bucketing, ZAR/NAD separation, and per-line rounding (net derived by subtraction) are all untouched. Re-asserted by test coverage across the all-time width. No money moves.

### Test gates

- **API:** 631 tests passing (earnings module 45 specs fully exercised: all presets + all-time + custom ranges + date validation + index queries)
- **Web:** 801 tests passing (both selector states, datepicker, pagination, UTC/local cap boundary, no sentinel in labels)
- **Lint:** `npm run lint:api` clean
- **Build:** `npm run build` clean, no warnings introduced

### Follow-ups

Both raised as v2 card options, not specced or queued for implementation — measure before picking either up.

- **[EDR-2](https://trello.com/c/4SOXI65Z)** — a composite `(status, createdAt)` index on `orders` would serve the actual earnings predicate (`status = 'delivered' AND createdAt BETWEEN`) better than the single-column index shipped here. Not a blocker; measure the query plan against real volume first.
- **[EDR-3](https://trello.com/c/D9yNimAK)** — push the earnings aggregation into SQL instead of `getMany()` + in-JS reduce (Open Questions Q2 above). Deliberately gated: this is a rewrite of well-tested money code, and should only be picked up once a real dataset shows the indexed scan isn't enough — not on suspicion.

## Related

- [[Vendor Earnings & Commission]] — the parent feature (VE-1 → VE-5)
- [[Money & Currency Rules]] — rounding, ZAR/NAD separation
- [[Analytics & Reporting]] — the sibling `from`/`to` query convention
- [[Order State Machine]] — `delivered` is the eligibility gate
- [[Listing Types & Vendor Rules]] — platform lines carry no commission
