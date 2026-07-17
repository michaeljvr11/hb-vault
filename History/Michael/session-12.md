---
operator: Michael
date: 2026-07-17
session: 12
tags: [ai-factory, ship-card, search, web]
---

# Session 12 — Search: admin synonyms UI (Angular)

## What we worked on
Trello card **MkRjnkvv** (#53) — "Search: admin synonyms UI (Angular)". Web-layer only. This card delivers the Angular admin screen for managing search synonyms (the UI atop the card #52 API endpoints), eliminating the need for direct API calls to manage domain synonyms without a deploy.

## What changed

**Admin service — `admin-search-synonyms.service.ts`:**
- Typed HttpClient wrapper: `list()`, `create(req)`, `update(id, req)`, `remove(id)`.
- Endpoints: `GET/POST /api/admin/search/synonyms`, `PATCH/DELETE /api/admin/search/synonyms/:id`.
- Consumes `@hb/shared` contracts (`SynonymDto`, `SynonymCreateRequest`, `SynonymUpdateRequest`); no duplicate DTOs.
- Location: `apps/web/src/app/core/api/`

**Admin page — `admin-search-synonyms` (standalone component):**
- Signals-based: `synonyms$`, `loading$`, `error$`, `selectedId$`.
- Typed reactive form with `FormArray` for equivalents (add/remove buttons, 20-item cap).
- Shared create/edit modal form; validation mirrors server DTO (term + equivalents required, max 100 chars each, no case/whitespace-sensitive duplicates).
- Per-row inline delete confirmation (SSR-safe, no `window.confirm`); state clears after delete.
- Load error surfaced as banner; refetch on retry.
- Hand-rolled HTML + DESIGN.md tokens + material-symbols, consistent with sibling admin pages (admin-users/admin-orders), not Angular Material.
- Location: `apps/web/src/app/features/admin/pages/admin-search-synonyms/`

**Routing & nav:**
- Route `admin/search-synonyms` under guarded admin block.
- New "Synonyms" nav item in `admin-shell`, positioned alongside admin-users/admin-orders.

**Client-side normalization:**
- Term + equivalents trimmed and lowercased on form submit to match server normalization.

## Key decisions

**Validation aligns with server DTO:**
- Term required, max 100; ≥1 equivalent; each equivalent required, max 100.
- Duplicates detected case/whitespace-insensitive.
- Errors surfaced as inline field messages, not top-level errors.

**FormArray for equivalents:**
- UI cap of 20 equivalents per synonym group (enforced client-side).
- Add/remove buttons within form array rows.

**No Material, existing idiom:**
- Hand-rolled HTML + DESIGN.md tokens (color, spacing, typography) + material-symbols icons.
- Matches admin-users/admin-orders for internal consistency; intentional divergence from Material to maintain the admin UI's cohesion.

**Inline SSR-safe delete:**
- Per-row delete confirmation rendered as inline `<button>` + confirmation state, no `window.confirm`.
- Works in server-side rendering without guards.

**Trim/lowercase normalization:**
- Client applies same normalization as server to ensure consistency in displayed data after round-trip.

## Test & review outcome

- **Web tests:** `npm run test -w @hb/web` → **446 passed** (service + page specs added; happy paths for load/create/edit/delete, validation errors, load-error surfacing, delete confirmation, equivalents array management).
- **Build:** `npm run build` → clean.
- **Code review:** code-reviewer (opus) — **SHIP**. No FAILs. One a11y nit (per-input `aria-label` on equivalents array rows) was identified and fixed in review.

## PR status
PR **#29** (https://github.com/michaeljvr11/hb-mono-repo/pull/29) — open, awaiting human merge. NEVER merged by AI; a human owns prod.

## Follow-ups

None. Synonyms admin UI is complete and ships immediately after PR merge. Related backlog item: card #51 (canonical-product / buy-box grouping) remains deferred to future phase.

## Orchestration

- **Single card, single layer:** web-only front-end (service + standalone component + route/nav).
- **Roles:** frontend-engineer (service + component + route), test-engineer (full Vitest coverage), code-reviewer (opus, SHIP with a11y polish).
- **No Agent Team:** straightforward UI card, no API/schema changes or cross-layer coordination.
- **Docs slice** (this session): `Product Search Engine.md` Implementation Notes section + this session log.
