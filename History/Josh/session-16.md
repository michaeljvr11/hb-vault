---
operator: Josh
date: 2026-08-04
session: 16
tags: [design, prototypes, prompt, repo-sync, retroactive]
---

# Session 16 — Front-end prototype ideas + designer brief; repo syncs

> Logged retroactively on 2026-11-01 from chat history. No PR. This was exploration plus housekeeping.

## Prototype ideas (2026-08-04)
- Explored several front-end visual directions and iterations based on the project plan
  (storefront only, no portals).
- Josh then asked for a **single prompt for another LLM that doubles as a BRS**. The first draft was
  eight pages and too long, so it was cut to one page covering: business context, the nine rules that
  constrain the UI, the screen inventory (storefront, vendor views, search, basket, checkout, profile),
  the tokens, and the deliverable.
- What the short version keeps, because a designer can't infer it: the SACU / ZAR–NAD peg / border
  facts, the multi-currency basket rule, the landed-cost table, and excluding the admin and vendor portals.
- What it drops: the full TypeScript contracts, per-screen state lists, persona detail, and the Tailwind config.

## Repo syncs
- **2026-08-05:** the analytics branch was already merged (PR #34). Switched to and pulled the base
  branch (+75 commits: analytics, customer profile, vendor branding, commission rates, pagination/sort).
  Three new migrations need `npm run migration:run`.
- **2026-09-07:** the base branch was 147 commits behind. Fast-forwarded it and pulled the vault
  (about 27 new Michael session logs and new specs such as shipping fee, reviews, wishlist and vendor
  profile). Dropped the local `.obsidian/workspace.json` to match the new gitignore.
- Both times, the local LAN dev overrides (`angular.json` `allowedHosts`, the `apiBaseUrl` in
  `environment.development.ts`) were stashed and restored. They are machine config, not commits.

## Follow-ups
- These LAN overrides keep getting stashed. A git-ignored local override file would remove the
  stash-and-restore step (also flagged in [[session-18]]).
