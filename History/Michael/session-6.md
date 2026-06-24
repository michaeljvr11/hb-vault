---
operator: Michael
date: 2026-06-24
session: 6
tags: [ai-factory, ship-card, storefront]
---

# Session 6 — Public storefront + restore SSR

## What we worked on
Trello card **kYbVZyqV** (#32) — "Make `/shop` public + restore SSR (remove authGuard, RenderMode.Server)". Slice 1 of 4 in the Public Storefront & SSR epic.

## What changed

**Frontend only:**
- `apps/web/src/app/app.routes.ts` — removed `canActivate: [authGuard]` from the `/shop` route. Anonymous visitors can now browse without redirect to `/login`.
- `apps/web/src/app/app.routes.server.ts` — flipped `shop` from `RenderMode.Client` to `RenderMode.Server` (enables SSR); corrected the stale comment that claimed "auth-protected routes must render on the client"; `/admin/**`, `/vendor/**`, and `/vendor` kept on `RenderMode.Client` (portal code depends on `localStorage`, which is absent on the server—no regression).

No `@hb/shared` contract changes. No API changes. Guard body deliberately untouched—returnUrl capture is card #33 (eOI6gKpN).

## Key decisions

**Single-layer frontend change:** no over-orchestration needed. Two files, one feature gate, one full vertical slice. Handled in one pass.

**Guard body left as-is:** the redirect logic still lives in `authGuard`; we only removed the route-level activation rule. The returnUrl attachment happens in the next card (#33). This separation keeps slices discrete.

**Evidence committed separately as chore:** after code review SHIP, the AI factory log was regenerated and committed as `chore(evidence): recompile AI-evidence report for kYbVZyqV` — decoupled from the feature commit so it doesn't muddy feature history.

## SSR safety

The full render chain is `isPlatformBrowser`-guarded:
- `Shop` component reads via `HttpClient` (interceptor attached).
- `NavBar` and `Footer` check `isPlatformBrowser` before DOM mutations.
- `AuthService.initialize()` (invoked in `APP_INITIALIZER`) guards token reads behind `isPlatformBrowser`.

Runtime verification:
- `GET /shop` (unauthenticated) → HTTP 200, full HTML with `ng-server-context="ssr"` marker, no `/login` redirect, no browser-API crash. Storefront SSR-rendered and browsable on server.

## Test & review outcome

- **Web tests:** `npm run test -w @hb/web` → 272/272 passed (incl. shop.spec.ts + auth-guard.spec.ts).
- **Build:** `npm run build` → clean SSR build, no errors.
- **Code review:** SHIP (code-reviewer). No FAILs.
  - Flagged pre-existing, out-of-scope follow-up: `shop.ts` `formatPrice` hardcodes `en-ZA` locale + a NAD/else-R branch. Worth a future storefront card under the epic (not introduced here, not a blocker).

## PR status
PR #19 — open, awaiting human merge.

## Remaining epic slices
- Slice 2 (eOI6gKpN): Capture returnUrl in guards on redirect.
- Slice 3 (vs3y5tDJ): Public `GET /vendors/:id` (approved-only filter).
- Slice 4 (EFXHykuT): Auth-aware storefront nav (sign-in link, account toggle, anonymous cart → login).

## Follow-ups
- Currency/locale formatPrice refactor (scope: `shop.ts`, `formatPrice()` fn, en-ZA hardcode + NAD fallback). Candidate for a future storefront UX card.

## Orchestration
- **Single slice, single layer:** frontend routing + render-mode only. No subagents needed.
- **Code review:** code-reviewer (opus) — SHIP.
- **Docs slice** (this session): Obsidian implementation note (`Public Storefront & SSR.md`) + this session log + evidence snapshot.
