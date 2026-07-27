# Money & Currency Rules

## Facts (code truth)

- Currencies: `ZAR` (ZA) and `NAD` (NA), ISO 4217. `COUNTRY_CURRENCY` maps each country to its home currency (`libs/shared/src/enums/country.ts`).
- **NAD is pegged 1:1 to ZAR but is stored as its own currency.** No FX conversion logic exists anywhere. If the peg ever changes it's a data/migration problem, not a schema rewrite.
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
