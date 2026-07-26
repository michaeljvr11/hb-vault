---
operator: Michael
date: 2026-07-26
session: 17
tags: [ai-factory, ship-card, vendor-portal]
---

# Session 17 — Vendor branding + profile-sections contract + schema

## What we worked on

Trello card **ZpvX9XIv** (VPC-1) — Vendor branding + profile-sections contract + schema + text read/write. Backend keystone for the Vendor Profile Customization feature; unblocks VPC-2..5. Delivered as a single backend-engineer session (no Agent Team needed; single-layer, text/JSON only — image upload deferred to VPC-2).

## What changed

**Shared contract (`@hb/shared`):**
- `VendorDto` extended with `logoUrl?`, `bannerUrl?`, `slogan?`, `profileSections?`.
- New `VendorProfileSection` interface: `{ id, title, type: 'curated' | 'category', productIds?, categoryId? }`.
- Promoted `VendorSectionType` to real enum (`CURATED` / `CATEGORY`, open for future `RULE`), not a union.
- New `VendorSelfDto` (owner self-view extends `VendorDto`, adds `website` + `description`) — solves spec gap: `GET /vendors/me` previously returned lean DTO, portal editor could not read its own editable fields.
- `UpdateVendorRequest` gained `slogan?` + `profileSections?`.
- New DTOs: `VendorProfileSectionDto`, `VendorSelfResponseDto`.
- Existing `VendorResponseDto` / `AdminVendorResponseDto` map branding; public shape still never leaks `registrationNumber` / `verificationDocumentUrl`.

**API (`apps/api/src/vendors/`):**
- `findByUserId()` + `GET /vendors/me` now return self-view (widened from public DTO).
- `VendorsService.update()` returns self-view shape for consistency.
- Security-critical guard: every `productId` in curated section must belong to acting vendor (batched query, `BadRequestException` on mismatch). Prevents cross-vendor feature.
- **Real bypass caught in review round 1:** category-typed section could carry unvalidated `productIds` past guard. Fixed via two custom `registerDecorator` validators (`IsCuratedProductIds`, `IsCategoryId`) enforcing discriminated union structurally: curated requires 1–24 v4-uuid productIds, forbids categoryId; category requires categoryId, forbids productIds.
- Service-level duplicate-section-id + duplicate-productId-in-section guards.

**Schema migration** (`1784332800000-VendorBrandingSections`):
- 4 nullable columns on `vendors`: `logoUrl varchar null`, `bannerUrl varchar null`, `slogan varchar null`, `profileSections jsonb null`.
- Symmetric up/down; no backfill.

**Caps confirmed** (resolved open question #2 in spec):
- Slogan ≤ 120 chars.
- Profile sections ≤ 10 per vendor.
- Curated section ≤ 24 productIds.

## Key decisions

1. **Promote `VendorSectionType` to enum, not union:** review feedback. Clearer intent + extensibility for future `RULE` member.
2. **Custom validators for discriminated union enforcement:** structurally verify curated ≠ category constraints; guard against async correctness gaps.
3. **Batched productId ownership query:** atomic check, not N+1 loop.
4. **Self-view DTO for `GET /vendors/me`:** restore portal editor's ability to read `website` + `description` (spec's noted gap), unblock VPC-3/4.
5. **Caps via CLARIFY → card owner confirmation:** slogan 120, sections 10, curated products 24.
6. **Stale id graceful degradation:** no categoryId re-validation at read time (VPC-5 render time drops dangling ids); noted as design choice, not a bug.

## Scope & orchestration

Single card, single backend-engineer session. Shared + API layers only (text/JSON contract + schema); multipart upload deferred to VPC-2. Unit test coverage on ownership + curated-products guard + persistence + self-view shape + boundary caps + cross-field validation.

## Test/review outcome

**Round 1 (FIX-FIRST):**
- Five issues found: (1) security bypass — category-type section validation; (2–5) correctness/coverage gaps — self-view shape consistency, duplicate guards, admin-edit-other-vendor, cap boundary edge-cases.
- All fixed + re-tested.

**Round 2 (SHIP):**
- All findings addressed; tests + lint + build green.
- Regression-tested ownership check (productId from wrong vendor rejected), curated ↔ category discriminated union enforced, self-view persisted correctly, caps enforced at boundary.

**Test pass rates:**
- API unit suite: all green (ownership guard, bypass path, persistence, self-view, caps, duplicate rejection).
- Lint: clean.
- Build: `shared→api` → passes.

## PR status

PR opened (link — see Trello card or lead-filled-in update). Branch: `feat/ZpvX9XIv-vendor-branding-schema`. Card moved to "In Review" pending human merge.

## Related notes

[[Vendor Profile Customization]] — Implementation Notes section added; "Open questions" row 2 (caps) marked resolved with actual numbers.

## Follow-ups

- **Unblocked:** VPC-2 (image upload endpoints), VPC-3 (portal text/slogan editor), VPC-4 (section builder), VPC-5 (public render).
- **Non-blocking advisories logged:**
  - Stale `profileSections` ids not re-validated at read time (degrade gracefully); candidate for future pass if UX signals a need.
  - `vendorId` could theoretically be undefined in ownership filter (unreachable code path); minor defensive guard candidate.
  - DTO duplication (`VendorResponseDto` + `VendorSelfResponseDto` + admin variant); candidate for future refactor, not urgent.
- **VPC-6 (rule-based auto sections):** remains backlog; needs sales-signal data model prerequisite.
