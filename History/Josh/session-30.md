---
operator: Josh
date: 2026-11-01
session: 30
tags: [ai-factory, ship-batch, site-qa, storefront, pdp, wishlist, a11y]
---

# Session 30 — QA-11: PDP wishlist button keeps its width

> One card of the `/ship-batch` storefront QA batch. Card [QA-11](https://trello.com/c/cPMDYtHG) `cPMDYtHG`, commit `a870542`, [PR #136](https://github.com/michaeljvr11/hb-mono-repo/pull/136) (branch `feat/yGRba8mk-storefront-qa-batch`). Spec: [[Site QA Findings Batch]]. The other cards in the batch: [[session-27]], [[session-28]], [[session-29]], [[session-31]].

## Problem
On the PDP the wishlist label swapped "Save to wishlist" → "Saved", so the button shrank and the `flex: 1` Add to Cart grew.

## What changed
- Both labels share one `inline-grid` cell. The inactive one is `visibility: hidden`, so the width follows the longer label and the hidden text stays out of the accessibility tree.
- `aria-pressed` and `aria-label` are unchanged. Icon-only below 768px is untouched. The product-card heart pop only uses `transform`, so it was left alone.
- The spec now asserts that only the current label is visible.

## Verification
Toggled with a real click at 1280, 768 and 375 (fixture customer): wishlist and Add to Cart widths unchanged, aria-pressed false → true. The batch's web tests and build are green, and code-reviewer returned SHIP.

## Follow-ups
- PR #136 waits for human review and merge. The card is in "In Review".
