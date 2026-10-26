---
operator: Michael
date: 2026-10-26
session: 60
tags: [ai-factory, ship-batch, storefront, shop-home]
---

# Session 60 — Shop Home Sections: SHS-6 shelves endpoint, SHS-7 filtered View all, radial nav scale

**Date:** 2026-10-26 · **Cards:** [SHS-6](https://trello.com/c/vEJI4mGQ) · [SHS-7](https://trello.com/c/5AhR9PHK) · [radial nav](https://trello.com/c/xXRUbiD5) · **Branch:** `feat/vEJI4mGQ-shop-shelves-endpoint` · **Status:** open, [PR #129](https://github.com/michaeljvr11/hb-mono-repo/pull/129)

- Shipped: one public `GET /products/shelves` replaces the client-side shelf derivations, with server-side dedupe and a 60s cache. `GET /products` gains four filters and two sorts. `/discover` reads the filters from the URL as chips, and the shelf "View all" links use them. Radial nav scales to the viewport.
- Decisions: one appearance per product per page. `q` with filters returns 400. Radial nav supports 320px, with labels ellipsized there until design decides.
- Tests: api 1905 (14 DB specs skipped, run locally), web 1781, lint clean, build clean.
- Follow-ups: 320px label treatment (design); DB-backed CI job; owner nod on the ~4 minute staleness window.
