---
operator: Michael
date: 2026-06-18
session: 1
tags: [ai-factory, ship-card, admin-portal]
---

# Session 1 — Admin portal shell + bootstrap endpoint

## What we worked on
Trello card **AP-6 / Z1ADCLQm** — "Admin portal — route area + shell + bootstrap endpoint (ship first)". Shipped the backend bootstrap flow and the full frontend shell to gate the admin surface.

## What changed

**Backend:**
- `POST /auth/bootstrap-admin` (`@Public`): one-shot endpoint to create the first admin. Counts existing admins via new `UsersService.countByRole`; returns **409 Conflict** if any admin already exists (self-sealing, cannot be re-exploited). On success, creates an admin user with `isVerified: true` (trusted setup action) and returns the standard auth response — access token + httpOnly refresh cookie identical to `/auth/login`. New shared contract `BootstrapAdminRequest`; `BootstrapAdminDto implements` it with class-validator `IsEmail` / `MinLength(8)` constraints. Password hashing defers to the `User` `@BeforeInsert` hook. Bruno request "Bootstrap Admin" added for one-click setup. Unit test enforces the 409 gate on second call, admin creation, and email collision (400).

**Frontend:**
- `/admin` route area: lazy-loaded, `canActivate: [authGuard, roleGuard]`, `data: { roles: ['admin'] }`. Unauthenticated and non-admin users are redirected to `/login` by existing guards.
- `AdminShell` component: standalone, signals-based, SSR-safe. Sticky top bar with H&B Market wordmark + "Admin Console" badge + sign-out. Sidebar nav: Dashboard / Vendors / Catalog / Users / Orders / Logs with active-link highlight. `<router-outlet>` for children.
- Six lazy placeholder child pages; default child redirects to `dashboard`.
- Styled to `admin-dashboard` Claude Design tokens. Three new tokens registered in `docs/design/DESIGN.md`: `--hb-secondary-fixed`, `--hb-on-secondary-container`, `--hb-scrim`.

**Schema:** no migration required — `users.role` and `users.isVerified` already exist.

## Key files
- `apps/api/src/auth/auth.controller.ts` — `bootstrap-admin` endpoint
- `apps/api/src/users/users.service.ts` — `countByRole` method
- `apps/api/src/shared/contracts/auth.ts` — `BootstrapAdminRequest` interface
- `apps/api/src/auth/dtos/bootstrap-admin.dto.ts` — validator DTO
- `apps/web/src/app/admin/admin.routes.ts` — route config + lazy children
- `apps/web/src/app/admin/admin.shell.component.ts` — shell component + nav
- `docs/design/DESIGN.md` — three new tokens

## Key decisions
- **Self-sealing 409 gate** — the endpoint refuses on any second call. Non-negotiable business rule, enforced by unit test.
- **Bootstrap admin has `isVerified: true`** — trusted setup action; exempt from the order-gating verification rule (which belongs to the orders module and is still a blocked card).
- **No migration needed** — both fields already exist from prior auth work.
- **Relative routerLink children under the shell** — each lazy page route uses `routerLink` relative to the `/admin` parent.
- **Tokens registered, not raw hex** — design tokens stored in `docs/design/DESIGN.md` so they're reused across screens.

## Test/review outcome
- **api 25/25** — all tests pass.
- **web 46/46** — all tests pass.
- **Lint clean**, **full build clean**.
- **Code review:** SHIP. No FAIL items. Design-token nits addressed.

## Follow-ups
- **AP-7 vendor approvals** is next — the keystone for the vendor portal (gating dependency).
- PR opened (link to be added on the card once merged).
