---
operator: Michael
date: 2026-06-19
session: 3
tags: [ai-factory, ship-card, vendor-portal]
---

# Session 3 — Vendor onboarding (become a vendor)

## What we worked on
Trello card **VP-5 / 1fqtc2JF** — "Vendor portal — onboarding 'become a vendor' flow". Scope shifted to full-stack mid-execution after discovering backend gaps. Shipped self-onboarding hardening across API and web in one PR.

## What changed

**Backend:**
- `POST /vendors` gatekeeper: changed from `@Roles(vendor)` (blocking customers) to `@Roles(CUSTOMER, VENDOR)` so an authenticated customer can self-apply.
- `VendorsService.create()` now sets `status: PENDING` (was auto-approved as APPROVED), grants the applicant `role = vendor` via `UsersService.update()` so they immediately access the role-gated `/vendor` portal, and throws **409 Conflict** on duplicate application (was 403).
- `VendorsModule` imports `UsersModule` — verified acyclic (no circular deps; `UsersModule` is a leaf importing only `TypeOrmModule.forFeature([User])`).
- No schema change, no migration — `vendors.status` defaults to `pending`, `users.role` already exists.

**Frontend:**
- New `/vendor/apply` route placed **before** the role-gated `/vendor` parent — unauthenticated and non-vendor users reach apply instead of being redirected.
- Standalone, signals-based `VendorOnboarding` component: reactive form with `businessName` (required), `tradingName` / `registrationNumber` / `countryCode` (optional); no document upload.
- Form submits `CreateVendorRequest` → `POST /vendors` → refreshes cached user (now vendor) → shows pending confirmation. A 409 surfaces "already applied".
- Vendors entering the apply flow get a status screen from `GET /vendors/me` with per-status messaging (pending / approved / rejected / suspended), with approved status linking through to `/vendor`.
- Added `AuthService.refreshCurrentUser()` helper for transparent user-state sync post-onboarding.
- New "Sell on H&B" nav link in the shop bar points to `/vendor/apply`.
- No `@hb/shared` contract changes — reused existing `CreateVendorRequest` / `VendorDto` / `VendorStatus` / `CountryCode`.

## Key decisions
**Role flip on apply:** A customer self-applying immediately becomes `role = vendor` with `status = pending`. Pending gates nothing real — public directory filters by approved status only, status transitions are admin-only, vendor-profile updates are owner-gated — so this is safe and unblocks immediate portal access.

**Token refresh:** `JwtStrategy.validate` re-fetches the user from the DB on every request, so the role flip takes effect immediately for API guards. The web client refreshes `/users/me` to sync its cached role for route guards.

**Import acyclicity:** `VendorsModule` → `UsersModule` → leaf (`TypeOrmModule.forFeature([User])`); unlike AP-9's `AdminModule` (which had to avoid importing `UsersModule` entirely), this import path is acyclic and safe.

## Scope shift (important)
The card opened "frontend only" but the backend `POST /vendors` was broken — auto-approving, role-gating to vendor (blocking customers), returning 403 instead of 409, and never granting the vendor role. Confirmed full-stack fix with the product owner in a clarifying message and shipped one PR across both layers.

## Test/review outcome
- **api 45/45** — all tests pass.
- **web 141/141** — all tests pass.
- **Lint clean**, **full SSR build pass** (only the two pre-existing SCSS budget warnings, unrelated).
- **Code review:** SHIP WITH NITS. Nits addressed in-PR: typed the test DTO to clear `no-unsafe-argument` warnings; made the 409 path load vendor status even if user refresh errors.

## Follow-ups
- Status-gate the vendor portal feature pages (dashboard/products/orders placeholders) by vendor status so a pending/rejected vendor can't access approved-only features (future card).
- `vendors.businessName` is `unique` — duplicate business name yields raw 500; friendly message is a deferred card.
- Document upload / KYC still deferred past v1.
- **This completes slice 5 of the vendor-portal sequence.**

## PR status
PR opening — to be moved to In Review (lead owns merge).
