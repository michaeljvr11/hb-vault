---
operator: Michael
date: 2026-11-01
session: 63
tags: [ai-factory, ship-batch, portals, storefront]
---

# Session 63 — Portal polish batch: QA-4, QA-6, QA-8, QA-17, QA-19

**Date:** 2026-11-01 · **Cards:** [QA-6](https://trello.com/c/gqE0a6e5) · [QA-17](https://trello.com/c/FH1uClnj) · [QA-8](https://trello.com/c/7qpJ5S5g) · [QA-4](https://trello.com/c/ZGEtbGcG) · [QA-19](https://trello.com/c/ZwHABBKv) · **Branch:** `feat/gqE0a6e5-portal-polish-batch` · **Status:** open, [PR #134](https://github.com/michaeljvr11/hb-mono-repo/pull/134)

- Shipped: shared centred `ModalDialog` for the vendor and admin product forms and the admin category form (QA-6), live fee table on /legal/shipping from a public read (QA-17), settlement table flex and nowrap fixes (QA-8), confirm-password field (QA-4), and always-visible vendor application answers in admin (QA-19).
- Decisions: the admin category form moved to the modal too, so the page does not mix a drawer and a modal. No unsaved-changes guard, because the old drawer had none.
- Tests: api 1926 passed (14 skipped), web 116 files / 1872 tests, lint:api clean, build clean.
- Follow-ups: "Where we deliver" copy on /legal/shipping needs a human decision (NA→ZA route); QA-1, QA-5, QA-7, QA-9, QA-11, QA-16, QA-18, QA-20 not started.
