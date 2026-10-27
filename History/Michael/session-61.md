---
operator: Michael
date: 2026-10-27
session: 61
tags: [ai-factory, ship-batch, storefront, checkout]
---

# Session 61 — Checkout QA batch: QA-12 saved addresses, QA-13 cart savings, QA-14/15 payment copy

**Date:** 2026-10-27 · **Cards:** [QA-12](https://trello.com/c/syUWbsqu) · [QA-13](https://trello.com/c/I6pewX9r) · [QA-14](https://trello.com/c/8wFo0GA9) · [QA-15](https://trello.com/c/YgW3VC7n) · **Branch:** `feat/I6pewX9r-checkout-qa-batch` · **Status:** open, [PR #132](https://github.com/michaeljvr11/hb-mono-repo/pull/132)

- Shipped: checkout saved-address picker (QA-12), server-computed cart and checkout savings via `listUnitPrice` and `discountTotal` (QA-13), one closed payment and trust disclosure (QA-14), and "You pay in X" currency copy built from `nadPerZar` (QA-15).
- Decisions: savings are per currency, never summed across currencies. The peg sentence says "1:1" only when the rate is exactly 1. QA-15 explains the existing currency rule and changes no money behaviour.
- Tests: api 1910 passed (14 skipped), web 1801 (111 files), lint:api clean, build clean.
- Follow-ups: SVO-9 (payment provider named in the security block); shared trust banner still says "ZAR or NAD, 1:1"; `checkout.scss` ~11 kB vs 8 kB budget.
