---
operator: Josh
date: 2026-10-15
session: 19
tags: [ship-card, defect, radial-nav, mobile, retroactive]
---

# Session 19 — Radial nav label defect (YS7PlgB6)

> Logged retroactively on 2026-11-01 from chat history and PR #121.

## What we worked on
Josh reported a defect: the Home and Search labels sat behind icons, Theme was off-centre, and the
other labels were too far away. Created the Trello card and fixed it on
`fix/YS7PlgB6-radial-nav-label-geometry`.
[PR #121](https://github.com/michaeljvr11/hb-mono-repo/pull/121), merged 2026-10-20.

## Root cause
- `.radial-nav__label` was anchored with a hard-coded 52px, the outer-ring size, but Home is 60px and
  Search is 58px.
- The Angular port dropped the design's `-50%` vertical centring, so every chip sat about 25px high.
- Labels had no `z-index`, so the next sibling item painted over them.
- Adding Theme took the outer ring from 4 to 5 items, which gave Theme the 84° slot. "Theme: System"
  ran off-screen at 375px.

## What changed
- Each chip is anchored off a per-item `--size` and centred on its icon, matching the design source.
- Explicit stacking: bands < items < labels.
- The visible chip just says "Theme". The `aria-label` keeps the full `Theme: Light/Dark/System`.
- Chips are capped to the space left in the viewport and ellipsize on an inner span, so the arrow survives.

## Test/review outcome
Measured in the dev server at 375×812: chip-to-icon centre offset went from about 25px to **0px**, all
7 chips sit a 12px gap from their icon, and nothing renders off-screen. Web tests: 1676 across 108 files.

## Follow-ups
- Below about 375px the outermost chip still can't fit, because the port dropped the design's responsive
  scale `k`. This was addressed in [[session-20]].
