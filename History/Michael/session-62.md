---
operator: Michael
date: 2026-10-28
session: 62
tags: [ai-factory, ship-batch, storefront, portals]
---

# Session 62 — QA polish batch: QA-2 icon subset, QA-3 flyout, QA-10 scroll, QA-21 commission rules

**Date:** 2026-10-28 · **Cards:** [QA-2](https://trello.com/c/eFktpP5m) · [QA-3](https://trello.com/c/NXIBdJ7c) · [QA-10](https://trello.com/c/m5WUCkuL) · [QA-21](https://trello.com/c/FQUHQkzC) · **Branch:** `feat/eFktpP5m-qa-polish-batch` · **Status:** open, [PR #133](https://github.com/michaeljvr11/hb-mono-repo/pull/133)

- Shipped: Material Symbols glyph subset with preload (QA-2, 4.0 MB to about 86 KB), hover-then-click fix for the category flyout (QA-3), `NavigationScroll` for top-on-new-page with query-only navigations kept (QA-10), and commission backdate and same-rate guards (QA-21).
- Decisions: the icon subset prunes GSUB ligatures before shaping, or closure pulls back all ~4,400 icons. QA-21 "same rate" checks the latest scheduled row, not the rate in force.
- Tests: api 1917 passed (14 skipped), web 114 files, lint:api clean, build clean.
- Follow-ups: QA-1, QA-4 … QA-9, QA-11, QA-16 … QA-20 not started; cold-cache Slow-4G check for QA-2 not run.
