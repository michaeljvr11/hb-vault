---
operator: Josh
date: 2026-11-01
session: 27
tags: [ai-factory, ship-batch, site-qa, storefront, shop, css]
---

# Session 27 — QA-1: Explore categories tiles: centred, no arrow

> One card of the `/ship-batch` storefront QA batch. Card [QA-1](https://trello.com/c/yGRba8mk) `yGRba8mk`, commit `997c0fd`, [PR #136](https://github.com/michaeljvr11/hb-mono-repo/pull/136) (branch `feat/yGRba8mk-storefront-qa-batch`). Spec: [[Site QA Findings Batch]]. The other cards in the batch: [[session-28]], [[session-29]], [[session-30]], [[session-31]].

## Problem
The `/shop` Explore categories tiles had an `arrow_forward` chevron and left-aligned text beside the icon.

## What changed
- `.category-card` became a centred column: icon, then name, then count. The chevron span and its SCSS, hover nudge included, are deleted.
- The waypoint dot, hover lift and `overflow-wrap` are kept. The skeleton tiles share the class, so they match.
- `arrow_forward` is still used in 8 other templates, so the committed icon subset needed no regeneration.

## Verification
Headless Chrome at 320, 375, 768 and 1280, light and dark: no chevron, icon and name centres line up with the tile centre, no overflow at 320px. A spec now pins the missing arrow. The batch's web tests and build are green, and code-reviewer returned SHIP.

## Follow-ups
- PR #136 waits for human review and merge. The card is in "In Review".
