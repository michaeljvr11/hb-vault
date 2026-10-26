---
operator: Michael
date: 2026-10-26
session: 59
tags: [ai-factory, ship-batch, storefront, shop-home]
---

# Session 59 — Shop Home Sections: SHS-3, SHS-4, SHS-5 shelves

**Date:** 2026-10-26 · **Cards:** [SHS-3](https://trello.com/c/3tGVSIlU) · [SHS-4](https://trello.com/c/mETkwXkj) · [SHS-5](https://trello.com/c/ABB1gwlR) · **Branch:** `feat/3tGVSIlU-shop-home-shelves` · **Status:** open, [PR #128](https://github.com/michaeljvr11/hb-mono-repo/pull/128)

- Shipped: On sale, Top rated, Made in Namibia, From South Africa and two category shelves on `/shop`, all derived client-side from the one existing products request by pure functions in `shop-shelves.ts` (max 8, min 4, in-stock only, omitted while loading or on error). Built on the `ProductShelf` from PR #127, which the owner merged first.
- Decisions: sequential, one commit per card, bundled at owner request because all three edit the same template block and derivation module. No `@hb/shared` or API change. No cross-shelf dedupe.
- Review caught a real bug: On sale ranked by `discountPercent`, which the API sends only to a product's owner or an admin; now ranks by `effectivePrice` against `price`. Fixtures must use the public payload.
- Tests: web 111 files / 1763 pass; build passes. Browser check used a throwaway mock API; the real API was not run.
- Follow-ups: server-side `onSale` filter and rating sort once the catalogue nears 100 products; the owner to review page length and the shelf order.
