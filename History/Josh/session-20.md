---
operator: Josh
date: 2026-10-27
session: 20
tags: [ship-card, radial-nav, mobile, accessibility, handedness, retroactive]
---

# Session 20 — Radial nav redesign: rings, labels under icons, hand switch (avwD5IcK)

> Logged retroactively on 2026-11-01 from chat history and PR #130.

## What we worked on
Follow-up card avwD5IcK on `feat/avwD5IcK-radial-nav-rings-handedness`.
[PR #130](https://github.com/michaeljvr11/hb-mono-repo/pull/130), merged 2026-10-27.
The first pass re-layered the nav to 3+4 rings with solved labels. Josh's review said it "still looks
terrible": labels should go underneath, spacing should be even, Home should sit centred on ring one
with Search to its right, and the left-hand option was hard to find. That led to a full redesign.

## What changed
- Labels hang centred **under** each icon.
- **Two-ring dial:** both rings sweep 13°–69°. The outer radius is 1.5× the inner, so every pair of
  neighbours is about 101px apart. Inner ring: Search · Home · Cart. Outer ring: Profile · Wishlist ·
  My Orders · Theme.
- Each ring is drawn as a thin rail between its first and last icon, so it never crosses a label.
- **An in-menu hand switch** in the opposite corner mirrors the nav and keeps it open. The toggle in
  profile settings (which needs sign-in) stays as a second way in.
- Fixed a positioning bug inherited from the port: items were anchored by their corner, 26px off their
  ring. They now sit in slots placed at their own centre.
- Robustness: labels are capped to a solved `labelBudget`, the whole fan scales below 360px, the scale
  is re-read when the menu opens, closed items are `inert`, the backdrop blur is 4px (down from 10px),
  the FAB has a clearer accessible name, and the badge text has fixed dark ink for contrast.
- **Tap fix (91416c7):** the redesign had dropped the `pointer-events` re-enable on open items, so real
  taps fell through to the scrim. Earlier checks used `element.click()`, which skips hit-testing, so
  they missed it.

## How it was verified
- The geometry was **solved, not nudged**: a constraint search over radius, sweep and tilt, judged
  visually in an interactive prototype artifact.
- Measured in the running app with headless Chrome over CDP at 320/360/375/412px, both hands. The
  tightest gap was 9.3px and no labels were truncated. Real touch and mouse events were then used.
- Web tests: 1800 across 112 files. A new spec asserts `pointer-events` open and closed, and was
  confirmed to fail without the fix.
- Merge conflicts with the base branch were resolved and the evidence was recompiled. The PR was left
  for a human to merge.

## Follow-ups
- Bottom safe-area inset for phones with a home indicator (needs its own card).
