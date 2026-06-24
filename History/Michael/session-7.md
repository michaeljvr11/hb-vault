---
operator: Michael
date: 2026-06-24
session: 7
tags: [ai-factory, ship-card, storefront, auth]
---

# Session 7 — Capture returnUrl in auth guards

## What we worked on
Trello card **eOI6gKpN** (#33) — "Capture returnUrl in authGuard / roleGuard on redirect to /login". Slice 2 of 4 in the [[Public Storefront & SSR]] epic. Wires the *producing* side of the returnUrl round-trip whose consuming side (Login/Register already read `returnUrl`) shipped earlier.

## What changed

**Frontend only — no `@hb/shared`, no API, no migration:**
- `apps/web/src/app/core/auth/auth-guard.ts` — redirect now `createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })`.
- `apps/web/src/app/core/auth/guards/role-guard.ts` — same `returnUrl` capture on the unauthenticated/forbidden redirect.
- `apps/web/src/app/core/auth/return-url.ts` (new) — `sanitizeReturnUrl()` open-redirect guard (same-app absolute paths only).
- `apps/web/src/app/auth/login/login.ts` + `register.ts` — `returnUrl` query param now sanitised before `navigateByUrl`; unsafe values fall back to default.
- Specs: `auth-guard.spec.ts`, `role-guard.spec.ts`, `return-url.spec.ts` (new), `login.spec.ts`, `register.spec.ts`.

## Key decisions

**Capture at the guard, sanitise at the consumer.** `state.url` is inherently a same-app path (`/…`), so the producing side is safe by construction; the open-redirect risk only exists where the param is read back off the URL bar and fed to `navigateByUrl`. So the single sanitiser chokepoint lives in `return-url.ts`, shared by both login and register — no duplicated logic.

**Threat model = the Angular router, not a full-page redirect.** `Router.navigateByUrl` never navigates cross-origin, so the vectors to block are protocol-relative (`//host`), backslash (`/\host`, browsers normalise `\`→`/`), and absolute/`javascript:` strings. All rejected; reviewer confirmed the exploitable forms are closed.

**Extended the settled guards, did not fork the auth flow** (per [[Auth & Roles]]).

## Test & review outcome

- **Web tests:** `npm run test -w @hb/web` → 285/285 passed (272 baseline + 13 new across the four guard/consumer specs).
- **Build:** `npm run build` → clean (pre-existing `shop.scss`/`admin-catalog.scss` budget warnings only, unrelated).
- **Lint:** `npm run lint:api` → clean.
- **Code review:** FIX-FIRST → SHIP. Blocking finding: `register.ts` had the sanitised navigation but no test → added two register cases mirroring login. NITs (sanitiser passes harmless oddities like `/@host`) confirmed non-exploitable via the router — left as-is.

## PR status
_(PR link added to the card on delivery)_ — open, awaiting human merge. NEVER merged by AI; a human owns prod.

## Remaining epic slices
- Slice 3 (vs3y5tDJ): Public `GET /vendors/:id` (approved-only filter).
- Slice 4 (EFXHykuT): Auth-aware storefront nav — now fully unblocked by this slice's `returnUrl` round-trip.

## Orchestration
- **Single slice, single layer:** frontend guards + consuming side. No Agent Team — implemented inline, one code-reviewer pass as the gate.
- **Code review:** code-reviewer (opus) — FIX-FIRST then SHIP after the register test was added.
- **Docs slice** (this session): `Public Storefront & SSR.md` Implementation Notes (Slice 2) + this session log.
