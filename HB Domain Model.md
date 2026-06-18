# HB Domain Model

Cross-border e-commerce & logistics platform: products sourced/listed in **South Africa (ZA)**, delivered into **Namibia (NA)**. Two business models share one codebase from day one — see [[Listing Types & Vendor Rules]].

## Entities (mirrors `libs/shared` + TypeORM entities)

| Entity | Key facts |
|---|---|
| User | role: `customer` / `vendor` / `admin` — see [[Auth & Roles]] |
| Vendor | belongs to a user; lifecycle `pending → approved` (or `rejected`/`suspended`) |
| Product | has `currency`, `originCountry`, `listingType`; `vendorId` nullable with DB CHECK |
| Category | product taxonomy |
| Address | tagged with `countryCode` (ZA/NA) |
| Cart / CartItem | per-customer staging for checkout |
| Order / OrderItem | snapshots `listingType` + `vendorId` per line; carries `currency`, `originCountry`, `destinationCountry` |
| Payment | behind `PAYMENT_PROVIDER` port (stub today) — see [[Money & Currency Rules]] |
| Shipment | `fromCountry`/`toCountry`/`customsReference`; customs is first-class — see [[Cross-Border & Customs]] |

## Invariants (enforced in schema/code — do not violate)

- Every record touching money or fulfilment carries explicit country + currency. The ZAR/NAD peg is **data, never an assumption**.
- One order can mix platform and vendor lines.
- State machines for orders/payments/shipments: [[Order State Machine]].

## Open business decisions (TBD — ask a human)

- Pricing rules: markup model, who sets vendor prices, promotional pricing.
- Cross-border delivery fee structure.
- Returns/refunds policy across the border.
- KYC requirements for vendor onboarding.
