# Cross-Border & Customs (ZA → NA)

The origin/destination pair on orders and shipments **is** the cross-border seam of the whole platform.

## Facts

- ZA and NA share a customs union (**SACU**) and a 1:1-pegged currency (ZAR/NAD). The peg is stored as data — see [[Money & Currency Rules]].
- Customs is a **first-class business step** even inside SACU: `ShipmentStatus` includes `at_border` and `customs_cleared`.
- Shipments carry `fromCountry`, `toCountry`, and a `customsReference`.

## Shipment lifecycle (code truth, `libs/shared/src/enums/shipment-status.ts`)

```
pending → booked → in_transit → at_border → customs_cleared → out_for_delivery → delivered
                                                              ↘ failed (from any active state)
```

## Rules for agents

- Never collapse the two countries into one "region" — every fulfilment record is explicitly tagged.
- `customsReference` is required before a shipment can leave `at_border` (business rule — enforce when implementing border logic).
- Courier integration goes through the `SHIPPING_PROVIDER` port only (stub today, deliberate). One adapter class + one module line to add a real courier.

## TBD (ask a human)

- Which courier(s) for the ZA→NA leg.
- Customs documentation requirements and who generates them.
- SLA per leg (domestic ZA, border, domestic NA).
