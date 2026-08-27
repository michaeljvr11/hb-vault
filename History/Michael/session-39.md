---
operator: Michael
date: 2026-08-27
session: 39
tags: [ai-factory, ship-batch, legal-compliance, consent]
---

# Session 39 — wokJ3PfW: Remaining legal pages + consent durability

**Date:** 2026-08-27 · **Cards:** LC-6, LC-7, LC-8, LC-9, LC-10 · **Branch:** `feat/wokJ3PfW-legal-pages-and-consent-durability` · **Status:** PR open

**Shipped:** Two final legal pages (`/legal/customs`, `/legal/vendor-agreement`, both `RenderMode.Prerender` — 10 prerendered routes now); footer's dead `href="#"` links wired to real routes; `users.termsAcceptedAt` column written in the same INSERT as the account, closing LC-3's best-effort consent gap; Google OAuth signup now captures consent through a `/accept-terms` interstitial gated in `authGuard`, backed by `POST /auth/accept-terms`. Vendor applications now require accepting the Vendor Agreement (`acceptedTerms` on `CreateVendorRequest`, `@Equals(true)`, new `vendor.terms_accepted` audit action); `AdminCreateVendorRequest` deliberately `Omit`s it — an admin creating a vendor row is not that vendor consenting.

**Code vs. spec (templates were wrong again):** the vault claimed the `customsReference`-before-leaving-`at_border` rule is "already enforced in the data model" — it is not: there is no shipping service, only the entity and a stub port, and the column is nullable. Orders carry `subtotal + shippingTotal + total` and no duty line anywhere, so the customs page states plainly that duty is not calculated or collected at checkout rather than implying a pass-through mechanism. No cancellation fee exists, so the vendor agreement says none is charged instead of inventing one. Onboarding's "Apply to become a **verified** vendor" was the KYC overstatement in miniature and is now honest.

**Decisions (all owner-approved before code):** durability via a `termsAcceptedAt` column over a shared transaction or `logOrThrow` — atomic by construction, and `logOrThrow` would throw after the user row was already committed. Explicitly overrides this spec's preference for reusing the audit log. OAuth consent via interstitial over implicit-at-creation (an "implied" record is weaker under POPIA/the ETA). Vendor agreement gated by a required checkbox rather than a link. Existing accounts get no DB backfill — a fabricated timestamp is worse than an honest null — so they are asked once at the interstitial on next use; the "not prompted" wording in the first LC-9 comment was corrected on the card after review caught it.

**Tests:** API 950 / Web 1052 · Lint clean · Build clean, 10 prerendered routes · Both new pages and the rewired footer verified live in the dev server. **Database verification (same day, after the PR opened, once Rancher was up):** the migration applied to the populated dev schema, reverted, and re-applied cleanly; no row was backfilled. End-to-end against the running API: register rejects missing/`false` `acceptedTerms` (400, no account created) and on success writes column and audit metadata with the same instant; a cleared column reports `null` and `POST /auth/accept-terms` stamps it `via: 'interstitial'`, idempotent, 401 unauthenticated; `POST /vendors` rejects missing/`false` consent and writes `vendor.terms_accepted`. Verification rows removed afterwards. The live `commission_rates` table holds one row at **15.00** — the published rate is confirmed, not assumed.

**Follow-ups:** LC-1 (entity facts) still blocks eleven placeholder tokens plus four new ones on the customs page and one on the vendor agreement. The terms gate is router-only — a caller holding an access token can still reach the API without an acceptance record; server-side enforcement is unbuilt. The vendor-side acceptance still rides the best-effort audit log alone — only the signup record got the durable column.
