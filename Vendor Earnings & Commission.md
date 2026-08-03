# Vendor Earnings & Commission

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[Money & Currency Rules]] · [[Order State Machine]] · [[Vendor & Admin Portals]] ·
[[Listing Types & Vendor Rules]] · [[Analytics & Reporting]] · [[HB Domain Model]].

Business-rules source: `H&B Brain` repo (sibling vault, business-plan/strategy — not this
vault, not `obsidian`-MCP-connected). Cited by file below; treat as settled fact, not TBD,
unless flagged as an open item there.

## Problem

[[Analytics & Reporting]] shipped vendor/admin revenue reporting as **gross GMV** and
explicitly deferred commission (its open question 4: *"Vendor revenue definition — gross
GMV (assumed) vs net of commission? Commission structure is unresolved... confirm before
[revenue reporting] ships numbers vendors will treat as settlement."*). [[Listing Types
& Vendor Rules]] and [[Money & Currency Rules]] both separately list the commission/fee
structure as TBD.

The business side has since settled this (`H&B Brain/08-revenue-model.md`,
confirmed 2026-07-14, service-fee % marked provisional): **15% commission, vendor keeps
85%**, and the rate is expected to change once operational costs are modelled — so it
must be **admin-configurable**, not hardcoded. `H&B Brain/13-payments-payouts.md`
(updated 2026-07-27) also defines *when* a line becomes payable — delivery plus a 48-hour
damage-claim window, not merely reaching `confirmed` — and a two-cadence reporting shape
(weekly-accruing balance vs bi-weekly settlement batch).

This note resolves those open questions and specs the reporting surface: admin sees
net earnings per vendor over preset windows; vendors see their own. **This is
visibility/reporting only — no money moves.** Actually executing a payout or integrating
a payment gateway is out of scope (see below).

## Business rules it must honour

From `H&B Brain` (cite on any implementation deviation):

- **`08-revenue-model.md`**: commission is **15%** of order (line) value at launch,
  vendor keeps 85%. Marked provisional — "subject to change once operational costs are
  modelled." This is *why* the rate must be admin-configurable, not a fixed constant.
- **`13-payments-payouts.md`**:
  - **Payout-eligibility trigger** = order line **delivered** AND its **48-hour
    damage-claim window has passed**. This is *not* the `confirmed` order status
    ([[Order State Machine]] — `confirmed` is the very first post-payment state, four
    transitions before `delivered`).
  - Vendor-visible balance **accrues weekly** (eligibility re-checked weekly); actual
    bank settlement batches **every two weeks**. These are two different numbers and
    both must be representable in the reporting surface — this note builds the
    reporting math, not the settlement job itself.
  - **"Held for vendors" is a distinct accounting line from day one** — customer money
    is not H&B revenue until the 15% fee is earned. Concretely: platform revenue
    (admin's real earnings) = sum of commission on eligible lines, *not* gross vendor
    line GMV like the current `AdminDashboardDto`/`VendorAnalyticsDto` split.
- **`10-marketplace-rules.md`**: vendor-caused cancellations carry a cancellation fee
  (amount **still TBD** upstream — do not invent a number). Cancelled/refunded lines
  never count toward earnings, accruing or settled.
- **`21-kpis-success-metrics.md`**: "Weekly payouts executed on time %" is a tracked
  KPI — the accruing/weekly figure this note specs is what that KPI would read from
  (the KPI itself, and any dashboard widget for it, is out of scope here).

From this vault (existing conventions this feature must not violate):

- **Money stays `numeric(12,2)` + explicit currency; ZAR/NAD never summed**
  ([[Money & Currency Rules]]). Every earnings figure carries `CurrencyCode`.
- **Order revenue = line-item snapshot at order time**, not recomputed from live
  prices ([[Analytics & Reporting]], [[Money & Currency Rules]]) — the commission rate
  applied to a line must be **snapshotted the same way**, so a later rate change can
  never retroactively restate a past line's earnings.
- **Vendor-scoped queries filter by ownership in the service layer**
  ([[Listing Types & Vendor Rules]], [[Vendor & Admin Portals]]) — a vendor sees only
  their own `order_items` lines, never another vendor's.
- **Any service method touching money gets a unit test in the same PR** — non-negotiable
  repo-wide, doubly so here (this *is* the accounting surface).
- **State transitions only through `OrdersService.updateStatus`**
  (`apps/api/src/orders/orders.service.ts:264`) — confirmed in code: `shipped →
  delivered` is **admin-only** ([[Order State Machine]]). This is the one and only
  place a `deliveredAt` timestamp can be stamped.

## Data model

Nothing in the current schema captures delivery time or a commission rate. Both are new:

1. **`deliveredAt` timestamp** — needed on the order (or per-shipment, per the "coupled
   state machines" note in [[Order State Machine]] — v1 assumption: order-level, since
   partial/mixed-line delivery is still a listed TBD there too). Stamped exactly once,
   in `OrdersService.updateStatus` on the `shipped → delivered` transition. Claim-window
   expiry is **derived** (`deliveredAt + 48h`), not stored — one named constant
   (`DAMAGE_CLAIM_WINDOW_HOURS = 48`), not a magic number scattered across services.
2. **Commission rate — effective-dated history, not a single mutable value.** An
   admin-editable *current* rate with no history would mean changing it today silently
   restates every past line's "earnings" retroactively — directly contradicts the
   "held for vendors is an accurate accounting line from day one" rule above. So:
   - An append-only `commission_rates` table (id, `percent`, `effectiveFrom`,
     `createdBy`, `createdAt`) — admin action *inserts* a new row, never edits/deletes
     an old one.
   - **`order_items` snapshots the rate that applied at order-creation time**
     (`commissionRatePercent numeric(5,2)` or similar), mirroring the existing
     `unitPrice`/`productName` snapshot pattern on that table. This is the anchor used
     for both the accruing and settled figures — a rate change never touches an
     existing line.
   - Platform (`listingType = 'platform'`) lines carry **no commission** (`NULL`) —
     there is no vendor to charge a fee to; see [[Listing Types & Vendor Rules]]'s
     "no fake house vendor" invariant. This is a real behavioural fork from the
     current `AdminDashboardDto` platform/vendor split, which the admin-earnings card
     must correct alongside adding commission.
3. **Rounding (confirmed 2026-07-28).** Per line: `commission = round_half_up(gross ×
   rate, 2dp)`; `net = gross − commission` (derived, never independently rounded).
   Rounding only one side and deriving the other guarantees `commission + net === gross`
   on every line — independently rounding both can drift by a cent (e.g. gross=R10.00,
   rate=15.55% → independently-rounded commission 1.56 + net 8.45 = R10.01, a cent over;
   deriving net instead gives 1.56 + 8.44 = R10.00 exactly). Rounding the commission side
   specifically means H&B computes and rounds its own cut and any sub-cent residue lands
   on the vendor's net, not on platform revenue — a deliberate, disclosed choice, not a
   neutral one.
4. **Bi-weekly settlement anchor (confirmed 2026-07-28) — placeholder, not a real ops
   decision.** `13-payments-payouts.md` never states which calendar date the two-week
   settlement periods count from. Since this note only builds reporting (no settlement
   job exists to actually anchor), implement one single named constant
   (`SETTLEMENT_ANCHOR_DATE` or equivalent) that all bi-weekly bucketing math reads from
   — not inlined/duplicated anywhere. Treat its value as provisional and trivially
   swappable; the real anchor is still a human ops decision for whoever eventually builds
   the settlement-execution job.

## @hb/shared contract impact

New `contracts/commission.ts` (or extend `contracts/order.ts` — decide at card time) and
extensions to `contracts/analytics.ts`/`contracts/vendor.ts`. Interfaces + enums only;
reuse `CurrencyCode`, `CurrencyTotalDto`, `OrderStatus` — do not duplicate.

- **`CommissionRateDto`** — `id`, `percent`, `effectiveFrom`, `createdAt`. Read model
  for the admin rate-history list.
- **`SetCommissionRateRequest`** — `percent` (validated range, e.g. 0–100),
  `effectiveFrom`. Admin-only endpoint input.
- **`VendorEarningsDto`** (admin, per-vendor) / **`MyEarningsDto`** (vendor, own) —
  windowed tallies: `accruing: CurrencyTotalDto[]` (delivered, in claim window or past
  it but not yet in a settlement batch) and `settled: CurrencyTotalDto[]`
  (bi-weekly-batch-equivalent), each scoped to a requested window
  `EarningsWindow = '1w' | '2w' | '1m'` (mixed semantics, confirmed 2026-07-28 —
  **`1w`/`2w` are rolling trailing periods ending now**, matching
  `AdminAnalyticsQuery`'s existing rolling-window convention, but **`1m` is the current
  calendar month** — 1st of the month through now, or the full month when a past month
  is targeted explicitly via `from`/`to`. Not a rolling 30 days. Explicit `from`/`to`
  always wins over the `window` preset when both are supplied), reusing the
  `AdminAnalyticsQuery`/`VendorAnalyticsQuery` `from`/`to` pattern from
  [[Analytics & Reporting]] rather than inventing a new query shape.
- Corrected **`AdminDashboardDto`** revenue split (or a new `AdminEarningsDto` —
  decide at card time so the existing dashboard contract isn't broken mid-migration):
  `platformCommission` (real H&B revenue — net, commission only) vs
  `platformListingGmv` (first-party platform-listing GMV, uncommissioned) vs
  `heldForVendors` (the accounting line the vault explicitly calls out).

Every new endpoint input is a class-validator DTO implementing the shared interface,
per repo non-negotiables.

## Out of scope (recorded so nobody assumes them)

- **Executing payouts or bank transfers.** `apps/api/src/payments/` is a stub by design
  ([[Money & Currency Rules]]) — this note specs what a vendor *would be owed*, never
  moves money.
- **Payment gateway integration** (Stitch / FNB Namibia eCommerce Switch / DPO —
  `H&B Brain/13-payments-payouts.md` still lists all three as unconfirmed with no
  published pricing).
- **Cancellation fee amount** — `10-marketplace-rules.md` open item, not resolved here.
  The mechanism (exclude cancelled lines from earnings) is in scope; the fee number
  itself is not.
- **Per-vendor negotiated/reduced commission rates** (Phase 1.5 subscription-tier
  discount mentioned in `08-revenue-model.md`) — v1 ships one global rate. Per-vendor
  overrides are a future card if/when subscriptions land.
- **A real refund flow.** No refund flow exists yet in code (`payment-status.ts`'s
  `refunded` state exists but nothing currently writes it) — building one is out of
  scope. The earnings computation **does** include the exclusion hook now (confirmed
  2026-07-28): a line is ineligible if its order is `cancelled`, or has any `payments`
  row with `status = refunded`. Currently inert (no writer exists yet) but wired so
  earnings are correct the moment a refund path lands, with no follow-up card needed.
- **Minimum payout amount** — `13-payments-payouts.md` lists this as unresolved.
- **The "Weekly payouts executed on time %" KPI dashboard widget itself** — this note
  builds the data the KPI would read, not the KPI tracking UI.

## Open questions

All resolved 2026-07-28 except one:

1. ~~**Rounding**~~ — **resolved**: per-line commission rounded half-up to 2dp, net
   derived by subtraction. See "Data model" point 3 above for the mechanics and why.
   [[Money & Currency Rules]]'s "rounding rules for vendor payouts and fees" TBD is
   answered for this feature specifically (still open more broadly for actual payout
   execution, which remains out of scope here).
2. ~~**"Last month" window**~~ — **resolved**: calendar month, not rolling 30 days.
   `1w`/`2w` stay rolling trailing windows. See "`@hb/shared` contract impact" above.
3. ~~**Bi-weekly settlement anchor**~~ — **resolved**: ship a placeholder anchor
   (single named constant, easily changed) rather than blocking on the real ops
   decision. See "Data model" point 4 above.
4. ~~**Refunds**~~ — **resolved**: the exclusion hook (any `payments` row with
   `status = refunded`) is built now, even though currently inert. See "Out of scope"
   above.
5. **Still open — `deliveredAt` granularity.** Order-level (v1 default, matches
   current order-level status) vs per-shipment/per-line, given [[Order State Machine]]
   already flags partial/mixed-line delivery as an open TBD independent of this
   feature. VE-3 ships the order-level default; revisit only if/when partial delivery
   is itself specced.

Contract file location (new `contracts/earnings.ts`) was settled during card-writing —
see VE-1/VE-3/VE-4/VE-5 below.

## Implementation Notes (VE-1)

**Date:** 2026-08-03  
**Card:** VE-1 (#70, `QAeB8YGv`)  
**Branch:** `feat/QAeB8YGv-admin-commission-rate` (PR open, not yet merged)

### What shipped

- **`libs/shared/src/contracts/earnings.ts`** (new): `CommissionRateDto`, `CreateCommissionRateRequest`, `CommissionRateListItemDto` (adds `inForce: boolean` flag for the active rate), `CommissionRateListDto`. Exported from `contracts/index.ts`.
- **`apps/api/src/commission/` module** (new): `CommissionRateEntity`, `CommissionRateService` (methods: `create`, `list`, `getRateAt(date)`), `CommissionController` with `GET /admin/commission-rates` and `POST /admin/commission-rates` (both `@Roles(ADMIN)`). No PATCH/DELETE routes — append-only by omission.
- **Migration** `1784419200000-CommissionRates.ts`: creates `commission_rates` table (id, `ratePercent numeric(5,2)`, `effectiveFrom timestamptz`, `note varchar(500)`, `createdByUserId`, `createdAt`). Adds a **unique index on `effectiveFrom`** to prevent concurrent-insert race. Seeds the 15.00% row from the earliest existing order's `createdAt` (or epoch if no orders exist).
- **Audit trail**: new `commission_rate.created` action logged via existing `AuditService`.
- **14 unit tests**: cover strictly-after boundary (409 Conflict on out-of-order or duplicate `effectiveFrom`), numeric-string→number coercion, `inForce` flagging, and `getRateAt` boundary resolution (`LessThanOrEqual` operator verified, not just mocked return).

### Key decisions for downstream cards

1. **Unique index on `effectiveFrom` for race prevention.** Initially the service had only app-layer read-then-write logic; code review flagged a race: two concurrent POSTs could both read the same "latest" row and insert conflicting rows. Fixed by adding a `UNIQUE` constraint on `effectiveFrom` in the migration and catching the resulting pg `23505` violation, rethrowing it as the same `ConflictException` an out-of-order sequential request gets. **This ensures the database, not the app, is the race-prevention gate.**

2. **`CommissionModule` is NOT `@Global()`**, unlike `AuditModule`. Initially built `@Global()` to mirror audit's pattern, but code review noted that `CommissionModule` owns an HTTP controller (audit does not) and currently has no cross-module consumer. Globalizing it would needlessly export HTTP surface into every module's injector. **VE-3 should explicitly `imports: [CommissionModule]` when it needs `CommissionRateService.getRateAt()` for order-item rate snapshots.** This keeps the module boundary clear.

3. **`getRateAt(date)` behavior**: boundary-inclusive (`effectiveFrom <= date`), throws (does not silently fall back) if no row covers the date. Should never happen post-seed, but VE-3 must not treat a thrown error here as "no commission" — it should propagate or be explicitly handled.

### Review & test outcome

- `npm run test:api`: 512/512 pass.
- `npm run lint:api`: clean.
- `npm run build`: clean (shared → api → web).
- Code-reviewer pass found 4 blocking issues, all fixed before PR:
  - Weak `getRateAt` test assertions (now verify the actual `LessThanOrEqual` query).
  - Missing unique constraint (added to migration).
  - Unbounded `note` length (limited to 500 chars via migration).
  - `CommissionController` not registered in public-route auth guardrail test (added).

### Scope explicit

- Out of scope: `orders`, `order_items`, payout eligibility, claim windows, settlement (VE-3/VE-4/VE-5).
- Out of scope: UI (VE-2).

## Implementation Notes (VE-2)

**Date:** 2026-08-03  
**Card:** VE-2 (#71, `WIZ4pJtk`)  
**Branch:** `feat/WIZ4pJtk-admin-commission-rate-screen` (PR #44 open, not yet merged)

### What shipped

- **`apps/web/src/app/core/api/commission.service.ts`** (new): `CommissionService` wrapping `GET /admin/commission-rates` and `POST /admin/commission-rates` endpoints from VE-1, with typed response mapping.
- **`apps/web/src/app/features/admin/pages/admin-commission/` component** (new, standalone Angular): reads and displays the current in-force commission rate as a headline; renders full effective-dated history as a sortable table; includes a "schedule new rate" form with client-side validation.
- **Admin route & nav**: new `/admin/commission-rates` route and "Commission" sidebar nav entry in `apps/web/src/app/app.routes.ts` and `apps/web/src/app/features/admin/admin-shell/admin-shell.ts`.
- **Form validation (client-side)**:
  - `ratePercent`: range 0–100, **at most 2 decimal places** (regex `/^\d+(\.\d{1,2})?$/` to avoid float-precision bugs — see Key decisions below).
  - `effectiveFrom`: optional `datetime-local` input; when blank, omitted from POST payload so server-side `now()` default applies; when unparsable, caught client-side (`Number.isNaN(parsed.getTime())`) before submission.
  - `note`: optional, max 500 chars (mirrors server's `@MaxLength(500)` constraint for fast-fail UX).
- **Error handling**: server-side 409 Conflict (out-of-order `effectiveFrom`) surfaced verbatim as an inline form error, not a raw stack trace or toast.
- **Double-submit guard**: form disabled during pending submission.
- **Vitest specs** (56 tests, 697 total across web): load tests (initial rate fetch, table render), submit success with auto-refresh, 409-conflict handling, double-submit guard, client-side validation (edge cases: 8.29%, 0.5%, invalid dates).

### Key decisions worth recording

1. **Float-precision validation gotcha (caught in code review).** Initial validation used `Math.round(rate*100) !== rate*100` to check 2dp. This fails for legitimate rates like 8.29% because `8.29*100 === 828.9999999999999` in IEEE 754 arithmetic, falsely flagging it as invalid. Fixed by checking the decimal-string representation directly with regex `/^\d+(\.\d{1,2})?$/`. **General lesson**: never validate fractional precision via float arithmetic; work on the string representation or use a decimal library if available in the stack.

2. **`effectiveFrom` omission from POST when blank.** Client-side leaves `effectiveFrom` out of the request body entirely (not a `null` field) when the form field is empty, so the server's `now()` default takes effect. Keeps the form optional without requiring client-side time logic.

3. **`effectiveFrom` parse error handling (caught in code review).** `datetime-local` input's `value` is parsed via `new Date(dateString)`, which silently returns `Invalid Date` for unparsable strings. Check `Number.isNaN(parsed.getTime())` before calling `submit()` to fail fast with a clear client-side message instead of letting the backend reject with a raw 400.

4. **Note length client-side mirror.** Server enforces `@MaxLength(500)` on `note`; adding the same check client-side (`note.length <= 500`) provides immediate feedback in the form UX without a round-trip, improving perceived responsiveness.

### Review & test outcome

- `npm run test -w @hb/web`: 697/697 tests passed (56 files).
- `npm run test:api`, `npm run lint:api`, `npm run build` (root): all clean.
- Code-reviewer flagged 4 blocking issues (all fixed before merge):
  1. **Float-precision validation bug** — initial `Math.round` check was imprecise. Fixed by switching to regex on the input string.
  2. **Untested decimal-precision branch** — test suite did not cover the 8.29% edge case. Added dedicated test.
  3. **Unguarded `effectiveFrom` parse** — `new Date()` can return `Invalid Date`; no check before `submit()`. Added `Number.isNaN()` guard.
  4. **Missing client-side note length check** — server enforces 500 chars; client had no mirror. Added.
- Reviewer also applied a small advisory simplification: merged two near-identical error-message signals into a single `submitError` property to reduce template boilerplate.

### Scope explicit

- Out of scope: backend order-item rate snapshots, payout eligibility (VE-3/VE-4/VE-5).
- Out of scope: vendor-facing earnings portals (VE-5).

## Vertical slices → Trello cards

Order: VE-1 → (VE-2 ∥ VE-3) → (VE-4 ∥ VE-5).

1. **VE-1** (#70, `QAeB8YGv`) — Admin-configurable commission rate: `@hb/shared`
   contracts + effective-dated `commission_rates` entity/migration + admin API.
2. **VE-2** (#71, `WIZ4pJtk`) — Admin portal commission-rate management screen.
   Depends on VE-1.
3. **VE-3** (#72, `trMZD1C5`) — Payout-eligibility data model (`deliveredAt` +
   derived claim-window) + net earnings computation service. Depends on VE-1
   (needs the rate snapshot to exist).
4. **VE-4** (#73, `1DAb9a1I`) — Admin cross-vendor earnings: `GET /admin/earnings`
   + UI. Depends on VE-3.
5. **VE-5** (#74, `WrPOgQDa`) — Vendor own-earnings: `GET /vendors/me/earnings` +
   portal screen. Depends on VE-3.
