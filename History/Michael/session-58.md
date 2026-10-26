---
operator: Michael
date: 2026-10-26
session: 58
tags: [ai-factory, ship-batch, storefront, shop-home]
---

# Session 58 — Shop Home Sections: spec, then SHS-1 hero + SHS-2 ProductShelf

**Date:** 2026-10-26 · **Cards:** [SHS-1](https://trello.com/c/Bn6dsEla) · [SHS-2](https://trello.com/c/lU68lYzv) · **Branch:** `feat/Bn6dsEla-shop-hero-height-and-product-shelf` · **Status:** open, [PR #127](https://github.com/michaeljvr11/hb-mono-repo/pull/127)

- Shipped: `/spec-feature` wrote the Shop Home Sections note and cards SHS-1..5 (Obsidian and Trello MCPs were down: vault written on disk, Trello via REST). Then `/ship-batch` for SHS-1 and SHS-2: reusable `ProductShelf` extracted from New in Namibia with no visible change, and a shorter `/shop` hero.
- Decisions: shelves derive client-side from the one `GET /products?limit=100`, no contract change; min 4 / max 8 per curated shelf; "Best sellers" excluded (no sales signal). Hero height was headline-driven, so the headline moved from `display` to `4xl` (800). Sequential mode, SHS-2 first because it moves styles out of `shop.scss`.
- Tests: web 110 files / 1730 pass; contrast 48 pairs pass; build passes with `INTERNAL_API_BASE_URL`. Hero measured in the browser pane at five widths; the products API was down so the loaded shelf is unit-tested only.
- Follow-ups: SHS-3 (On sale, Top rated), SHS-4 (origin shelves), SHS-5 (optional category shelves). Human to eyeball the 4xl headline; desktop hero cut is 19-28%, a little past the spec.
