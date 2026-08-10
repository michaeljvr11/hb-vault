# Session 27 — WL-1…WL-5: Product wishlist

**Date:** 2026-08-10 · **Card:** WL-1 `rtBV85cQ` · WL-2 `MiiSqOS5` · WL-3 `RrgIdrIo` · WL-4 `zG9zNAh8` · WL-5 `L54YO6fI` · **Branch:** `feat/rtBV85cQ-product-wishlist` · **PR:** [#49](https://github.com/michaeljvr11/hb-mono-repo/pull/49) · **Status:** in review

- Shipped: `wishlist_items` table + migration, `GET/POST/DELETE /wishlist` API, signal-backed `WishlistService`, heart toggle on `ProductCard` (shop/discover/vendor-profile/PDP-related), PDP "save to wishlist" toggle, standalone `/wishlist` page, radial-nav + desktop nav-bar entries with badge (desktop icon-only, no count).
- Decisions: radial-nav actually renders on 4 templates, not just `/shop` — spec's out-of-scope claim was wrong, corrected in the spec note; desktop badge stays icon-only per the spec's resolved open question 4, overriding WL-5's stale card text; only `WishlistService.reset()` added to logout, `CartService`'s pre-existing gap left as a follow-up; wishlist migration reviewed but not run (Docker unavailable).
- Tests: api 606/606, web 777/777, lint clean, build clean; review 0 FAIL, follow-up audit closed hydration-gate + unique-violation-race coverage gaps.
- Follow-ups: run the `WishlistItems` migration against a live DB; `CartService.reset()` still not called on logout (pre-existing gap, left alone).
