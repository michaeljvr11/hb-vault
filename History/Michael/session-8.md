---
operator: Michael
date: 2026-06-26
session: 8
tags: [ai-factory, ship-card, storefront, public-api]
---

# Session 8 — Public vendor profile endpoint (approved-only)

## What we worked on
Trello card **vs3y5tDJ** (#34) — "Make GET /vendors/:id public (approved-only) for SSR vendor profiles". Slice 3 of 4 in the [[Public Storefront & SSR]] epic. API-only; enables SSR-rendered vendor-profile pages to fetch vendor data without a token.

## What changed

**API only — no `@hb/shared`, no migration:**
- `apps/api/src/vendors/vendors.controller.ts` — added `@Public()` to `@Get(':id')` so SSR can fetch without a token. Route ordering already safe — `directory`, `me`, `me/dashboard` declared before `:id`.
- `apps/api/src/vendors/vendors.service.ts` — `findOne` now queries `{ where: { id, status: VendorStatus.APPROVED } }`. Non-approved (pending/rejected/suspended) vendors hit `NotFoundException` → 404, matching `findDirectory` visibility. Response stays on lean `VendorResponseDto` (no PII leak).
- `apps/api/src/vendors/vendors.service.spec.ts` — new `describe('findOne (public vendor profile)')`: approved vendor returned & mapped to public DTO; PII fields omitted; each non-approved status → 404 (parametrised); unknown id → 404. Mock honours the `where` clause so the filter is genuinely exercised.

## Key decisions

**Approved-only filter in the service (query layer).** Keeps data-access logic centralised; reusable if a future admin route needs a separate scoped lookup.

**No `@hb/shared` change, no migration.** All fields already exist; this is a visibility + filtering change.

**Public DTO enforced by existing response-mapping flow.** The controller returns `VendorResponseDto`, never `AdminVendorResponseDto`, so `registrationNumber` / `verificationDocumentUrl` are never leaked.

## Test & review outcome

- **API tests:** `npm run test:api` → 112 passed (+6 new in vendors.service.spec.ts).
- **Web tests:** `npm run test -w @hb/web` → 285 passed (no web files touched).
- **Build:** `npm run build` → clean (shared → api → web).
- **Lint:** `npm run lint:api` → clean.
- **Code review:** SHIP. No FAILs. One non-blocking note: if an admin single-vendor-detail view is ever needed it must be a separate admin-scoped route (this one returns the lean DTO and hides non-approved) — out of scope.

## PR status
_(PR link added to the card on delivery)_ — #21, open, awaiting human merge. NEVER merged by AI; a human owns prod.

## Remaining epic slices
- Slice 4 (EFXHykuT): Auth-aware storefront nav (sign-in link, account toggle, anonymous cart → `/login?returnUrl=…`).

## Orchestration
- **Single slice, single layer:** API-only, no dependencies. Implemented inline — no Agent Team.
- **Code review:** code-reviewer (opus) — SHIP.
- **Docs slice** (this session): `Public Storefront & SSR.md` Implementation Notes (Slice 3) + this session log.
