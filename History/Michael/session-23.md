# Session 23 — VE-2: Admin commission-rate management screen

**Date:** 2026-08-03  
**Card:** VE-2 (#71, shortLink `WIZ4pJtk`, board "H&B E-commerce")  
**Feature:** Vendor Earnings & Commission (UI layer, second slice of 5)  
**Branch:** `feat/WIZ4pJtk-admin-commission-rate-screen`  
**Status:** PR open, not yet merged

## What was built

The admin-facing UI layer for rate management: a standalone Angular component showing the current in-force commission rate as a headline, rendering the full effective-dated history as a table, and providing a form to schedule new rates. This card ships VE-2 and completes the rate-management vertical slice (VE-1 + VE-2); VE-3 through VE-5 handle order-item snapshots, payout eligibility, and vendor-side earnings visibility.

**No backend changes.** VE-2 is purely frontend, building on the VE-1 API contracts and endpoints (`GET /admin/commission-rates`, `POST /admin/commission-rates`).

## Changes

**Repo:** hb-mono-repo  
**Branch:** `feat/WIZ4pJtk-admin-commission-rate-screen`

### 1. Commission API service (`apps/web/src/app/core/api/commission.service.ts`)

**New file:**
- `CommissionService` wraps the two VE-1 endpoints.
- `getCommissionRates()` — calls `GET /admin/commission-rates`, returns `CommissionRateListDto`.
- `createCommissionRate(req: CreateCommissionRateRequest)` — calls `POST /admin/commission-rates`, returns `CommissionRateDto`.

### 2. Admin commission page component (`apps/web/src/app/features/admin/pages/admin-commission/`)

**New component (standalone Angular):**
- `AdminCommissionComponent` with three main sections:

  1. **Current rate headline** — displays `commissionRateList.items.find(r => r.inForce)?.ratePercent` formatted as "Current: X.XX%".
  
  2. **History table** — Material table showing all rates, sortable by `effectiveFrom`, columns: effective date, rate (%), created by, created date, notes. Automatically refreshes after successful POST.
  
  3. **Schedule new rate form:**
     - `ratePercent` (required, range 0–100, at most 2 decimal places).
     - `effectiveFrom` (optional `datetime-local` input; server defaults to `now()` if omitted).
     - `note` (optional, max 500 chars).
     - Submit button, double-submit guard (button disabled during pending request).
     - Inline error display for all three fields.

### 3. Routing & navigation

**Updated files:**
- `apps/web/src/app/app.routes.ts` — new route `{ path: 'commission-rates', component: AdminCommissionComponent }` under the `admin` lazy route.
- `apps/web/src/app/features/admin/admin-shell/admin-shell.ts` — added "Commission" nav entry to the admin sidebar, linking to `/admin/commission-rates`.

### 4. Client-side validation

**Form validators:**
- `ratePercent` — Angular FormControl with custom validator that checks range (0–100) and exactly 2 decimal places.
  - **Validation method:** regex `/^\d+(\.\d{1,2})?$/` on the string representation, not float arithmetic (see Key decisions below).
  - Rejected values: 8.299%, 10.555% (too many decimals), 101% (out of range), non-numeric input.
  - Accepted values: 15, 8.29, 0.5, 99.99.
  
- `effectiveFrom` — optional; if provided, parsed via `new Date(value)` with `Number.isNaN()` guard to catch unparsable dates before submission (see Key decisions below).
  
- `note` — optional, max 500 chars (mirrors server's `@MaxLength(500)` for client-side UX).

**Error feedback:**
- Server 409 Conflict (`effectiveFrom` out of order or duplicate) surfaced as `submitError` in the template, not a raw exception.
- All client-side validation errors show inline near the field (via Material error container).

### 5. Tests

**56 Vitest specs** covering:

**Component lifecycle:**
- Initial load: fetches rates, populates table and current-rate headline.
- After successful submit: clears form, refreshes the rates table, no error signal.

**Form submission:**
- Minimal request (only `ratePercent`) — `effectiveFrom` omitted from POST body, server defaults to `now()`.
- Full request (`ratePercent`, `effectiveFrom`, `note`) — all three sent; server validates.
- Server 409 Conflict — displays error message (e.g., "This effective date is before the latest rate") as `submitError`.

**Client-side validation:**
- Decimal precision: 8.29% accepted, 8.299% rejected, 8.2% accepted.
- Range: 0% and 100% accepted, 101% rejected, -1% rejected.
- `effectiveFrom` unparsable date — guard catches `Invalid Date` before submission, displays "Invalid date" error.
- `note` length — form submits with full 500-char note, rejection tested (server-side, not duplicated here).

**Double-submit guard:**
- Button disabled during pending request; re-enabled on success or error.
- Second click while request in flight has no effect.

**Table rendering:**
- Displays all rates, sorted by `effectiveFrom` descending.
- Current rate (highest `effectiveFrom`) marked visually (e.g., bold, icon, or badge).

**Test suite:**
- `npm run test -w @hb/web`: 697/697 tests passed (56 files, includes these 56 new specs for VE-2).

### 6. Edge cases & error scenarios

- **Empty form submission** — form requires `ratePercent`, so submit is blocked until a value is entered.
- **Duplicate `effectiveFrom`** — server rejects with 409; client displays the error.
- **Out-of-order `effectiveFrom`** — e.g., `2026-08-05` when the latest rate is `2026-08-06`; server rejects with 409; client displays the error.
- **Network failure** — request timeout or connection error shows generic error (not wired in this pass, but the error boundary exists).

## Key decisions worth recording

### 1. Float-precision validation: regex on string, not arithmetic

**What:** Form validation checks if a number has at most 2 decimal places using regex `/^\d+(\.\d{1,2})?$/` on the input *string*, not float arithmetic.

**Why:** Initial attempt used `Math.round(rate*100) !== rate*100`. This fails for legitimate rates like 8.29%:
```
8.29 * 100 === 828.9999999999999  (IEEE 754 floating-point error)
Math.round(828.9999999999999) === 829
Math.round(828.9999999999999) !== 828.9999999999999  // true, incorrectly flags as invalid
```

**How:** Parse the input as a string, apply regex `/^\d+(\.\d{1,2})?$/`, reject if no match. This avoids float-precision ambiguity entirely.

**Lesson:** When validating fractional precision (especially 2dp for money/percentages), work on the string representation or use a decimal library. Float arithmetic can introduce false negatives. Apply this pattern to any future 2dp validation (currency amounts, interest rates, etc.) in the codebase.

### 2. Omit optional `effectiveFrom` from POST body when blank

**What:** When the `effectiveFrom` form field is empty, it is **not included** in the request body (not sent as `null` or `undefined`).

**Why:** VE-1's POST endpoint defaults `effectiveFrom` to `now()` if omitted. Sending `null` would require the backend to interpret it as "use server time"; omitting it keeps the intention clear and lets the server's default apply.

**How:** In the submit handler, only add `effectiveFrom` to the request object if the parsed date is valid (post-guard check). If the field is empty, the property is simply not set.

**Impact:** Users can schedule a rate for "effective immediately" (now) by leaving the datetime field blank, or specify a future date. No guessing at intended behavior.

### 3. Parse `datetime-local` safely before calling submit

**What:** `datetime-local` input's `.value` can be an unparsable string. Parse it to `Date`, check `Number.isNaN(parsed.getTime())` before submitting.

**Why:** The HTML `datetime-local` type doesn't enforce a format at the HTML level (it depends on the browser). Passing an invalid string to the backend results in a raw 400 error. Catching it client-side provides clear UX.

**How:**
```typescript
if (this.form.get('effectiveFrom').value) {
  const parsed = new Date(this.form.get('effectiveFrom').value);
  if (Number.isNaN(parsed.getTime())) {
    this.form.get('effectiveFrom').setErrors({ invalidDate: true });
    return; // don't submit
  }
}
```

**Impact:** Users see "Invalid date" inline, not a generic 400 from the server.

### 4. Mirror server-side constraint: note ≤ 500 chars

**What:** Form validation limits `note` to 500 characters (same as server's `@MaxLength(500)`).

**Why:** Server enforces the limit, so enforcing it client-side catches the error immediately in the form UX, avoiding a round-trip 400.

**How:** Add `maxlength="500"` to the HTML input and sync the FormControl validator, or check manually in the submit handler. Material's `matInput` with `maxlength` handles it visually; the validator confirms programmatically.

**Impact:** Better perceived responsiveness; users see feedback before trying to submit a 600-char note.

### 5. 409 Conflict surfaced as inline form error, not a toast or stack trace

**What:** Server responds `409 Conflict` with a message like `{ error: "This effective date is before or equal to the latest existing rate" }`. This is displayed in the component as `submitError` and rendered inline in the template, not as a separate toast or console error.

**Why:** 409 is a validation error (not a system failure), specific to the rate-scheduling domain logic. It belongs in the form context where the user can correct the date and retry, not in a transient notification.

**How:** Service catches the 409 and includes the error message in the response (or error handler extracts it from the response body). Component sets `submitError = message` and re-enables the form for retry.

**Impact:** Users can immediately see what went wrong (e.g., "This effective date must be after 2026-08-03") and try a different date without dismissing a notification or refreshing the page.

## Review & test outcome

**Code review pass** flagged 4 blocking issues, all fixed before PR:

1. **Float-precision validation bug** — `Math.round(rate*100) !== rate*100` incorrectly rejected 8.29%. Fixed: switched to regex on string input. Verifies all edge cases (8.29%, 0.5%, 15%, 100%).

2. **Untested decimal-precision branch** — the 8.29% edge case was not covered by tests. Fixed: added dedicated test case verifying acceptance of 8.29% and rejection of 8.299%.

3. **Unguarded `effectiveFrom` parse** — `new Date(value)` can return `Invalid Date`. No check before submission. Fixed: added `Number.isNaN(parsed.getTime())` guard; test verifies a malformed date string is caught and shows an error inline.

4. **Missing client-side note length check** — server enforces 500 chars, but client had no validation. Fixed: added `maxlength="500"` and FormControl validator; test verifies a 600-char note is rejected with "Note must not exceed 500 characters".

**Advisory (applied, not blocking):**
- Simplified error signalling: merged two separate error fields (`dateError`, `genericSubmitError`) into a single `submitError` to reduce template boilerplate and keep error state consistent.

**Test results:**
- `npm run test -w @hb/web`: 697/697 pass.
- `npm run test:api`: 512/512 pass (VE-1 tests, unaffected).
- `npm run lint:api`: clean.
- `npm run build` (root): clean (shared → api → web).

## What's not in scope (out-of-scope clarifications for this card)

- Backend order-item rate snapshots (VE-3).
- Payout eligibility, claim windows, settlement (VE-3/VE-4/VE-5).
- Vendor-facing earnings portals (VE-5).
- Permission/audit logging UI (audit logs exist, but displaying them is not part of VE-2).

## Follow-ups

### Next in chain (unblocked by this card):

- **VE-3** (#72, `trMZD1C5`) — Payout-eligibility data model (`deliveredAt` + claim-window) + net earnings computation. **Depends on VE-1** for `CommissionRateService.getRateAt()`. Blocks VE-4 and VE-5.

### No open items from this card:
- Float-precision handling documented (general lesson for future 2dp validation in the codebase).
- Client-side form validation mirrors server-side constraints (no drift).
- Double-submit guard prevents duplicate submissions.

## PR link

Trello card 71: https://trello.com/c/WIZ4pJtk  
Branch: `feat/WIZ4pJtk-admin-commission-rate-screen`  
PR #44: (link pending, PR open but not yet merged as of session date)
