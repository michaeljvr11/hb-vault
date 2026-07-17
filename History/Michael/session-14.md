---
operator: Michael
date: 2026-07-17
session: 14
tags: [ai-factory, ship-batch, customer-profile]
---

# Session 14 — Customer Profile batch

## What we worked on
Trello cards **eq7ybOyS, U9C8764Y, 1StR8MKR, xkD93Anr, uGvvSURk** — Customer Profile slices 1–5, bundled in a single `/ship-batch` PR. Shipped user profile self-service surface (edit profile + change password), full address book CRUD, and customer order history read-only view.

## What changed

**Backend (`apps/api/src/users` + `apps/api/src/addresses`):**
- `PATCH /users/me` — update `firstName`/`lastName` only. Role/email/isActive/isVerified withlisted on DTO + ValidationPipe + explicit mapping; never reach repository. New `@hb/shared` contract `UpdateProfileRequest`.
- `PATCH /users/me/password` — change password. Requires `currentPassword`; validates bcryptjs hash (cost 12) before rewrite. Reuses `UsersService.setPassword()` → hashes + refreshes session + clears tokens = logged out everywhere including current. New contract `ChangePasswordRequest`.
- Addresses CRUD (GET/POST/PATCH/DELETE /addresses) — ownership via `{id, userId}` lookups, 404-not-403 (mirrors `OrdersService.findOneForUser`). Create forces caller's userId. UpdateAddressDto explicit fields (no `@nestjs/mapped-types` deliberate no-new-dep).
- No schema change, no migration. Existing `users` and `addresses` entities already have all required columns.

**Frontend (`apps/web/src/app/features/profile/`):**
- New `/profile` route, authGuard only (no roleGuard — all authenticated roles). Standalone, signals-based `ProfileShell` mirroring `VendorShell` pattern with nav sections: Details/Orders/Addresses.
- Details section: reactive form → `PATCH /users/me` + `AuthService.refreshCurrentUser()`. Change-password form → `PATCH /users/me/password`, inline server error, then "Password changed — sign in again." + `setTimeout(1800ms)` → `AuthService.logout()`.
- Orders section: read-only list/detail in-component (admin-orders pattern). Uses `formatPrice` with order's own currency; ZAR/NAD never summed. No mutations.
- Addresses section: list/add/edit/delete with inline delete confirm + error banner. Shared `extract-error-message.ts` normalises NestJS string|string[] error messages.

## Key decisions

1. **Password change invalidates all sessions** (owner confirmed 2026-07-17). Current session included, user is forced to re-sign-in. Spec assumed lower-risk; confirmed both name edits (no re-auth) and password changes (total logout) with owner.
2. **`/profile` reachable by all authenticated roles** — customers, vendors, admins. Vendors keep `/vendor/profile` for business profile. Confirms vendors are users underneath.
3. **No default-address flag** — open question left open, out of scope. Requires schema + checkout redesign, not requested.
4. **Ownership checks in service layer** — `findOneForUser` pattern on addresses, mirrors orders (404-not-403).

## Scope & orchestration

- Five cards (eq7ybOyS, U9C8764Y, 1StR8MKR, xkD93Anr, uGvvSURk) bundled via `/ship-batch` into one branch/PR.
- Sequential slices with parallel gates where disjoint: 2 parallel backend agents (users + addresses) → 1 shell agent (resumed once after connection drop) → 2 parallel section agents (details, orders, addresses components) → test-engineer → code-reviewer.
- 7 commits total: 9f57ae7 (users endpoints), b2dd003 (addresses CRUD), 3c72b4c (profile shell+details), 8d9b579 (order history), 4ca3de8 (address book), e78420d (review WARN fixes), 08cd8f5 (evidence recompile).

## Test/review outcome

- **API:** 356/356 tests pass (100%).
- **Web:** 513/513 tests pass (100%).
- **Lint:** clean.
- **Build:** full SSR build clean.
- **Test-engineer audit:** no coverage gaps; all money/inventory/order logic tested.
- **Code-reviewer:** SHIP, 0 FAIL / 3 WARN (all fixed in e78420d):
  - SCSS hardcoded colors → `color-mix` over `--hb-primary`/`--hb-error` tokens.
  - Shared error-message normaliser added (`extract-error-message.ts`).
  - Address lineTotal spec assertion tightened.

## PR status

PR #33 — https://github.com/michaeljvr11/hb-mono-repo/pull/33 — open, awaiting human merge (lead owns prod).

## Evidence snapshot

After `npm run evidence` (2026-07-17):
- **128 commits** (2026-06-11 → 2026-07-17)
- **118** carry AI-authorship trailer
- **75** specs compiled
- **17** prod blocks (guardrails)

## Follow-ups

- Default-address flag + checkout UX (separate card, schema change required).
- Profile picture / avatar upload (deferred; no file-storage provider wired).
- Address book optional fields hardening (e.g., postal code format per country — deferred polish).

## Deliverables (this session)

- **Customer Profile.md** Implementation Notes section — what shipped, resolved open questions, orchestration, follow-ups.
- **History/Michael/session-14.md** (this file) — batch summary, test/review outcome, PR link.
- **AI Factory — Evidence Log.md** — refreshed headline figures (128 commits · 118 AI-tagged · 75 specs · 17 prod blocks) + PR #33 entry.
