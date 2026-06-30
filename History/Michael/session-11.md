---
operator: Michael
date: 2026-06-30
session: 11
tags: [ai-factory, ship-card, categories, api]
---

# Session 11 — Safe category delete (block in-use / parent categories)

## What we worked on
Trello card **Sm1kSNO8** (#39) — "Safe category delete — block in-use / parent categories (API)". API only. This card closes a data-integrity gap: admins cannot silently corrupt the taxonomy by deleting an in-use or parent category.

## What changed

**API guard — `CategoriesService.remove`:**
- Now loads category with `products` and `children` relations before deletion.
- Refuses deletion with HTTP 409 `ConflictException` if either is non-empty:
  - Products: "Cannot delete a category that has products; reassign them first"
  - Children: "Cannot delete a category with subcategories; delete or reparent them first"
- Unknown id still returns 404 (via `findOne` null-check, replacing old `delete().affected===0` path).
- Only an empty category is deleted via `repository.remove(category)`.
- Location: `apps/api/src/categories/categories.service.ts`

**Admin UI error surfacing — `admin-catalog`:**
- `deleteCategory` captures `HttpErrorResponse` and sets `categoryDeleteError` signal (`err.error?.message` with generic fallback).
- 409 message rendered as `error-banner` in categories section.
- Error clears when a new delete is confirmed.
- Category row is preserved on 409 conflict.
- Location: `apps/web/src/app/features/admin/pages/admin-catalog/`

**Test coverage:**
- `apps/api/src/categories/categories.service.spec.ts` — 4 new cases: empty delete, 404 unknown id, 409 blocked-by-products, 409 blocked-by-children.
- `apps/web/src/app/features/admin/pages/admin-catalog/admin-catalog.spec.ts` — 2 new specs: 409 error surfacing, stale-error clear on new confirm.

## Key decisions

**Block with 409, no cascade or auto-reassign:**
- v1 enforces explicit intent: to delete a parent, subcategories must be deleted or reparented first. To delete an in-use category, products must be reassigned first.
- Bulk reparent / "move products to…" tooling is a future admin card.

**Service-layer guard, no controller change:**
- Guard applied in `remove`, transparent to the controller.

**No `@hb/shared` change, no migration, no schema change:**
- Swap from `delete(id)` to `findOne` + `remove`; the contract and schema are unchanged.

## Test & review outcome

- **API tests:** `npm run test:api` → **130 passed** (+4 new in categories.service.spec.ts).
- **Web tests:** `npm run test -w @hb/web` → 295 passed (+2 new in admin-catalog.spec.ts).
- **Build:** `npm run build` → clean.
- **Lint:** `npm run lint:api` → clean.
- **Code review:** code-reviewer (opus) — **SHIP**. No FAILs. One forward-looking follow-up: `remove` hydrates full `products`/`children` collections to check emptiness; could switch to count/exists checks if category product volume grows (unlikely to block, but worth tracking for high-volume scenarios).

## PR status
PR **#24** (https://github.com/michaeljvr11/hb-mono-repo/pull/24) — open, awaiting human merge. NEVER merged by AI; a human owns prod.

## Follow-ups

- **Query optimization:** switch to count/exists checks instead of full collection hydration if category product volume grows.
- **Reassign tooling:** bulk product reassignment and category merge/move are future admin UX cards, separate from the safety gate.

## Orchestration

- **Single card, single layer:** API + admin-UI front-end, no `@hb/shared`/migration. Service guard + UI error surfacing, straightforward composition.
- **Roles:** backend-engineer (implemented service guard), frontend-engineer (error surfacing), test-engineer (full coverage), code-reviewer (opus, SHIP).
- **No Agent Team:** tight scope, single delivery vehicle.
- **Docs slice** (this session): `Category Taxonomy & Discovery.md` Implementation Notes section + this session log.
