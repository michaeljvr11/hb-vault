---
operator: Josh
date: 2026-11-01
session: 31
tags: [ai-factory, ship-batch, site-qa, legal, contact, ssr]
---

# Session 31 — QA-16: WhatsApp contact is a real link

> One card of the `/ship-batch` storefront QA batch. Card [QA-16](https://trello.com/c/pf0T6ggh) `pf0T6ggh`, commit `a53ccea`, [PR #136](https://github.com/michaeljvr11/hb-mono-repo/pull/136) (branch `feat/yGRba8mk-storefront-qa-batch`). Spec: [[Site QA Findings Batch]]. The other cards in the batch: [[session-27]], [[session-28]], [[session-29]], [[session-30]].

## Problem
The returns and terms pages printed `wa.me/264…` as plain text.

## What changed
- New `CONTACT_DETAILS.whatsappUrl`, built from the one `PHONE_E164`. `SOCIAL_LINKS.whatsapp` now points at it (review nit, `ef840fc`).
- Both pages render `<a target=_blank rel="noopener noreferrer">WhatsApp +264 81 355 9921</a>`. The per-page `whatsappNumber` regex derivations are removed.
- No other plain-text `wa.me/` mentions remain.

## Verification
Specs on both pages assert the href, target, rel and text. The prerendered `/legal/returns` and `/legal/terms` HTML in the build output contain the anchor. The batch's web tests and build are green, and code-reviewer returned SHIP.

## Follow-ups
- PR #136 waits for human review and merge. The card is in "In Review".
