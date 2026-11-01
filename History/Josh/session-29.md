---
operator: Josh
date: 2026-11-01
session: 29
tags: [ai-factory, ship-batch, site-qa, storefront, nav, currency, css]
---

# Session 29 — QA-9: Nav no longer shifts on ZAR/NAD

> One card of the `/ship-batch` storefront QA batch. Card [QA-9](https://trello.com/c/WR6Extzq) `WR6Extzq`, commit `4f2eeab`, [PR #136](https://github.com/michaeljvr11/hb-mono-repo/pull/136) (branch `feat/yGRba8mk-storefront-qa-batch`). Spec: [[Site QA Findings Batch]]. The other cards in the batch: [[session-27]], [[session-28]], [[session-30]], [[session-31]].

## Problem
Toggling the header currency changed the button width ("ZAR" and "NAD" differ in proportional Inter), so the search bar and icons moved.

## What changed
- `.nav-bar__currency-label` is `inline-block; min-width: 4ch; text-align: center`, wider than either code.
- QA-15 (PR #132) kept the visible label as the bare code, so 4ch fits. The switcher only shows from 1280px.

## Verification
Real mouse toggles NAD → ZAR → NAD at 1280 and 1440: all 45 visible nav element boxes identical, label 36.9px in both currencies. The batch's web tests and build are green, and code-reviewer returned SHIP.

## Follow-ups
- PR #136 waits for human review and merge. The card is in "In Review".
