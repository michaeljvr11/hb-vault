---
operator: Josh
date: 2026-06-19
session: 3
tags: [ai-factory, ship-card, vendor-portal]
---

# Session 3 — Vendor Portal Foundation (VP-1 / EXPtZCJL)

## What we worked on
Card VP-1 "Vendor portal — route area + layout shell" — lazy-loaded `/vendor/**` route area, role-gated via existing `authGuard` + `roleGuard`, standalone `VendorShell` component mirroring the admin portal foundation, and four placeholder pages.

## What changed

**Frontend only** — no backend changes:
- Uncommented and wired the `/vendor` lazy route in `apps/web/src/app/app.routes.ts`: `canActivate: [authGuard, roleGuard]`, `data: { roles: ['vendor'] }`, `loadComponent: () => VendorShell`. Non-vendor and unauthenticated users are redirected to `/login` by the existing guards.
- New standalone signals-based `VendorShell` component at `apps/web/src/app/features/vendor/vendor-shell/` — sticky top bar (H&B Market wordmark + "Vendor Portal" badge + sign-out), sidebar nav (Dashboard / Products / Orders / Profile with active-link highlight), `sidebarOpen` signal for toggle/close, `<router-outlet>` for children. SSR-safe, no browser APIs.
- Four lazy placeholder child pages under `apps/web/src/app/features/vendor/pages/` (dashboard, products, orders, profile) — each returns "Coming soon" so routes resolve without errors.
- Full Vitest spec matching `admin-shell.spec.ts`: renders, validates nav labels/paths, verifies sidebar toggle/close, confirms signOut → `AuthService.logout()`.
- Design tokens from `docs/design/DESIGN.md`, no hardcoded colors.

## Key decisions

- Single frontend-engineer subagent — single-layer card, no Agent Team needed.
- Mirror AP-6 (admin shell) pattern exactly: same component structure, same guard logic, same test shape. Reuse = consistency.
- Placeholder pages prevent 404s and allow later VP cards to swap in real screens without route changes.
- All role-gating via existing guards + route metadata — no new guard code.

## Test & review outcome

- `npm run test -w @hb/web` 72/72 pass (includes existing + new VendorShell spec).
- `npm run lint:api` clean.
- `npm run build` (SSR) full pass.
- Code review verdict: **SHIP** (no blocking issues).

## Follow-ups

- **VP-2:** vendor dashboard wired to a vendor-summary read-model endpoint (`GET /vendors/me/dashboard` or similar).
- **VP-3 / VP-4:** real product management and orders/fulfilment screens (deferred; placeholders ready).
- **VP-5:** vendor onboarding UI — "become a vendor" → `POST /vendors` flow.
- Real screens replace placeholders as cards ship; no route changes needed.

**Branch:** `feat/EXPtZCJL-vendor-portal-shell` (single commit). **PR:** open, awaiting review.
