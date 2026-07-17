Related: [[Auth & Roles]] · [[HB Domain Model]] · [[Order State Machine]] · [[Money & Currency Rules]]

## Problem

Customers can browse, checkout, and (per [[Auth & Roles]]) "view own orders/addresses" —
but there is no self-service surface for any of it. `GET /users/me` exists (returns
`UserDto`) but there's no update endpoint, no password change, and the `addresses` module
is still a domain skeleton (entity + module wiring, **no controller/service**). `GET /orders`
and `GET /orders/:id` already work (`OrdersController`) but nothing in the web app surfaces
them to a customer. Compare [[Vendor & Admin Portals]], which already ships this exact
shell pattern (`VendorShell` with lazy `dashboard/products/orders/profile` routes) for the
vendor role — the customer profile should follow the same structural pattern, not invent
a new one.

## Scope

**Backend**
1. `PATCH /users/me` — update `firstName`/`lastName` only. Email change is **out of
   scope** (see open questions — re-verification implications unresolved). `role`,
   `isActive`, `isVerified` are never client-settable (settled rule, [[Auth & Roles]]).
2. `PATCH /users/me/password` — change password. Requires `currentPassword` +
   `newPassword`; verify against the existing `bcryptjs` hash before rewriting. No email
   notification (out of scope — no email provider wired yet beyond the existing
   verify/reset token flow).
3. Address book CRUD, owner-scoped, on the existing `addresses` module/entity:
   - `GET /addresses` — list the caller's addresses.
   - `POST /addresses` — create (reuses `CreateAddressRequest`).
   - `PATCH /addresses/:id` — update; 404 (not 403) if the address isn't the caller's,
     matching the ownership-check pattern already used in `OrdersService.findOneForUser`.
   - `DELETE /addresses/:id` — delete; same ownership scoping.
4. Order history: **no backend work** — `GET /orders` / `GET /orders/:id` already exist
   and already scope to the authenticated user.

**Frontend** (`apps/web`)
5. `/profile` route (customer-facing, `authGuard` only — any authenticated role can view
   their own profile, not just `customer`). Standalone signals-based shell mirroring
   `VendorShell`'s nav pattern, with sections/tabs: Details, Orders, Addresses.
6. Details section: view + edit `firstName`/`lastName` (calls `PATCH /users/me`) and a
   change-password form (calls `PATCH /users/me/password`).
7. Orders section: list the customer's orders (`GET /orders`) with status + total, and a
   detail view (`GET /orders/:id`) — read-only, no status-transition UI (that's
   fulfilment-side, out of scope here).
8. Addresses section: list/add/edit/delete via the new address endpoints (slice 3).

## `@hb/shared` contract impact

New interfaces in `libs/shared/src/contracts/`:
- `UpdateProfileRequest { firstName?: string; lastName?: string }`
- `ChangePasswordRequest { currentPassword: string; newPassword: string }`
- Reuses existing `AddressDto` / `CreateAddressRequest` for the address book (no new
  address contract needed — the module just needed a controller).

No changes to `OrderDto` / `AdminOrderListItemDto` etc. — order history is read-only reuse.

## Out of scope

- Email address change / re-verification flow.
- Account deactivation or self-delete.
- Default/preferred address flag (doesn't exist on `Address` today — would be a schema
  change; not requested, flagged as an open question below).
- Order actions from the profile (cancel, reorder, return/refund) — no return/refund
  policy exists yet ([[HB Domain Model]] open decisions).
- Notification preferences, saved payment methods (no payment provider wired — ports +
  stub only).

## Open questions (ask a human before building)

- Should changing `firstName`/`lastName` require re-entering the password, or is a plain
  authenticated PATCH sufficient? (Password change clearly needs `currentPassword`; profile
  detail edits are lower-risk — assumed no extra auth needed unless told otherwise.)
- Is a "default address" flag wanted for checkout convenience, or is address selection at
  checkout out of this card's scope entirely (checkout flow untouched here)?
- Should `/profile` be reachable for `vendor`/`admin` roles too (they're also `User` rows),
  or should vendors be pointed at their existing `/vendor/profile` instead to avoid two
  profile surfaces? Assumed: yes, generic `/profile` works for any authenticated role,
  since a vendor is still a customer underneath — confirm this doesn't conflict with
  future vendor-profile plans ([[Vendor & Admin Portals]] follow-ups).


## Implementation Notes (2026-07-17)

**Branch:** `feat/eq7ybOyS-customer-profile`  
**PR:** https://github.com/michaeljvr11/hb-mono-repo/pull/33  
**Cards:** eq7ybOyS, U9C8764Y, 1StR8MKR, xkD93Anr, uGvvSURk (5 slices, 1 bundled batch)

### What shipped

**Backend** — Five parallel + sequential slices. Users endpoints + Addresses CRUD on the previously skeleton module.
- `PATCH /users/me` — update `firstName`/`lastName` only. Role/email/isActive/isVerified undeclared on the DTO + whitelist ValidationPipe + explicit field mapping prevents them reaching the repository (as spec required).
- `PATCH /users/me/password` — change password. Requires `currentPassword`; validates against existing bcryptjs hash (cost 12) before rewriting. Reuses `UsersService.setPassword()` internally. On success: **invalidates ALL sessions including the current one** — spec decided with owner 2026-07-17 (see "Resolved open question 2" below). Pre-hashes, clears refresh tokens, forces logout everywhere.
- Addresses CRUD (GET/POST/PATCH/DELETE /addresses) — full ownership validation via `{id, userId}` lookups (404-not-403, mirrors `OrdersService.findOneForUser` pattern). Create forces caller's `userId`. UpdateAddressDto declares fields explicitly (no `@nestjs/mapped-types` — deliberate dependency call).

**Frontend** — Signals-based shell + three child sections.
- New `/profile` route (authGuard only, no roleGuard — all authenticated roles can view their own profile; vendors keep `/vendor/profile` for business profile; see "Resolved open question 3" below). Standalone `ProfileShell` mirroring `VendorShell` structure, with nav sections: Details/Orders/Addresses.
- Details section: reactive form for `firstName`/`lastName` (calls `PATCH /users/me` + refreshes user), change-password form (calls `PATCH /users/me/password`, shows inline error on failure, then "Password changed — sign in again." then `setTimeout(1800ms)` → `AuthService.logout()`).
- Orders section: read-only list + detail split-view (reuses existing `GET /orders` endpoints; no new mutations). Formats price with order's own currency (ZAR/NAD never conflated); mirrors admin-orders in-component pattern.
- Addresses section: list/add/edit/delete with inline delete confirm + error banner (admin-catalog pattern). Uses shared `extract-error-message.ts` to normalize NestJS string|string[] messages.

**Test/review outcome:**
- API: 356/356 tests pass. Web: 513/513 tests pass. Lint clean, build clean.
- Test-engineer audit: no coverage gaps, all money/inventory/order logic covered.
- Code-reviewer verdict: **SHIP**, 0 FAIL / 3 WARN, all three WARNs fixed in commit e78420d (SCSS literals → `color-mix` over tokens; shared error normaliser; lineTotal spec assertion).

### Resolved open questions

1. ✓ **Profile re-authentication (question 1):** Name edits do NOT require re-entering password (assumed lower-risk). Confirmed with owner — this shipped as-is.
2. ✓ **Password-change session invalidation (question 1 variant):** Password change **does** invalidate ALL sessions including the current one (reuses `setPassword` → refresh token cleared). Confirmed with owner 2026-07-17; UI explains then forces sign-out at 1800ms timeout.
3. ✓ **Role scope of `/profile` (question 3):** `/profile` is reachable by **all authenticated roles** (customer/vendor/admin), not role-gated. Vendors access their own-user profile here; their business profile stays at `/vendor/profile`. Confirms vendors are still users underneath; no conflict with future vendor-profile plans.

### Remaining open questions

- **Default/preferred address flag (question 2):** Out of scope — deferred. Requires schema change + checkout flow redesign; not requested, flagged as open for future work.

### Orchestration

- `/ship-batch` with 5 cards bundled (1 branch, 1 PR).
- 2 parallel backend agents → 1 shell agent (resumed once after connection drop) → 2 parallel section agents → test-engineer → code-reviewer.
- 7 commits: 9f57ae7 (users endpoints), b2dd003 (addresses CRUD), 3c72b4c (profile shell+details), 8d9b579 (order history), 4ca3de8 (address book), e78420d (review WARN fixes), 08cd8f5 (evidence recompile).

### Follow-ups

- Default-address flag + checkout integration (separate card).
- Profile picture / avatar upload (deferred, no file-storage provider wired).
