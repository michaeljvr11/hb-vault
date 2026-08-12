# Session 28 — IP59Crue–BP5q32o4: Storefront UI cleanup batch

**Date:** 2026-08-12 · **Cards:** UIC-1 `IP59Crue` · UIC-2 `7sclIgtI` · UIC-3 `BP5q32o4` · **Branch:** `feat/IP59Crue-storefront-ui-cleanup` · **Status:** PR open, awaiting merge

- Shipped: Removed dead nav items ("SME Directory", "Logistics") + redundant "SME Verified" badges across 8 components; centralised snackbar config in `NotificationService` (19 call sites, bottom placement, Material token fixes); global `box-sizing: border-box` reset + PDP sticky-bar max-width sizing to eliminate site-wide horizontal overflow.
- Decisions: Bundled three slices into one PR per spec's rationale (UIC-1's SCSS/call-site deletions would conflict with UIC-2/3 if parallel); Material token bug (correct tokens: `--mat-snack-bar-*` not `--mdc-snackbar-*`) was inert, causing unreadable transparent surface — fixed in UIC-2.
- Behaviour: `/discover` now shows all products (platform + vendor) by default; was hiding half the catalogue via `smeOnly = true` toggle removed by UIC-1.
- Tests: web 782/782 green, build clean (pre-existing SCSS budget warnings unrelated). Blast radius measured benign: admin/profile/vendor sidebars, auth cards, synonym rows all reflow correctly, zero clipping. Pre-existing 375px/768px overflow (nav-bar actions + discover grid) measured not a regression, flagged for follow-up.
- Follow-ups: New card for 375px/768px viewports; design audit (exports still carry "SME Verified" badges, divergent from implementation).
