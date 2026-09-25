# Money & Currency Rules

## Facts (code truth)

- Currencies: `ZAR` (ZA) and `NAD` (NA), ISO 4217. `COUNTRY_CURRENCY` maps each country to its home currency (`libs/shared/src/enums/country.ts`).
- **NAD is pegged 1:1 to ZAR but is stored as its own currency.** Display-only FX conversion logic now exists (cart/checkout/order history remain settled in listed currency; browse surfaces show converted prices with explicit on-cart note when display ≠ settlement; see Implementation Notes below). If the peg ever changes it's a data/migration problem, not a schema rewrite.
- Money columns: Postgres `numeric(12,2)` + an explicit currency column, always. Never floats, never an implied currency.
- Payments go through the `PAYMENT_PROVIDER` port (logging stub today, deliberate — no real provider). Provider config is server-side only; **no payment keys in frontend env files, ever**.

## Payment lifecycle (`libs/shared/src/enums/payment-status.ts`)

```
pending → authorized → paid
        ↘ failed          ↘ refunded
```

## Rules for agents

- Any service method touching money gets a focused unit test in the same PR. Non-negotiable.
- Never hardcode `ZAR`; resolve currency from the country context.
- Order totals: store amounts as recorded at order time (snapshot), don't recompute from live product prices.

## TBD (ask a human)

- Payment provider choice — stale candidate list corrected 2026-07-28: current shortlist
  per `H&B Brain/13-payments-payouts.md` (updated 2026-07-27) is **Stitch** (front-runner
  but Namibia support now looks unlikely — no public evidence it operates there),
  **FNB Namibia eCommerce Switch**, and **DPO Group** (licensed Namibian facilitator);
  none has confirmed Namibian pricing yet. Payfast/Paystack are not on the current
  shortlist.
- Rounding rules for vendor payouts and fees — proposed default in
  [[Vendor Earnings & Commission]] (per-line commission rounded half-up to 2dp,
  `net = gross − commission`); still needs human confirmation.
- Refund flow across the border (currency of refund = currency of payment, presumably — confirm).

Related: [[Cross-Border & Customs]] · [[HB Domain Model]]

## Implementation Notes — 2026-09-25 (prelaunch batch, bNZweqNF)

**Currency switcher shipped display-only** (owner decision 2026-09-25, card bNZweqNF). Browse surfaces (product card, PDP, search suggestions, wishlist) now convert via `convertPrice()` using exact BigInt cents × micro-rate with half-up rounding (reviewer verified 4.8M test cases, 0 mismatches). Cart/checkout/order history keep settling in the product's listed currency with an explicit on-cart note when display ≠ settlement (suppressed for mixed-currency carts). Peg rate is now **data**: `platform_settings.nadPerZar numeric(10,6) NOT NULL DEFAULT 1` (migration `1788518400000-PlatformSettingsNadPerZar`), public `GET /api/settings/currency` → `PublicCurrencySettingsDto { nadPerZar }`, admin-editable via `PATCH /api/admin/settings` (both `notificationEmails` and `nadPerZar` independently optional; explicit null → 400; empty body → 400; audit logs `nadPerZar {from,to}`). Web: `DisplayCurrencyService` (root scope); default = country of signed-in user's most recently saved address via `COUNTRY_CURRENCY`, else **NAD** (owner: most customers are Namibian); explicit choice persisted in localStorage; re-resolves on sign-in/out via `switchMap` keyed on user id (cancels stale in-flight fetch). Rate fetch failure or non-positive/non-finite rate → no conversion, never assumed. PR pending.
