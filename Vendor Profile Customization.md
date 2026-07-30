# Vendor Profile Customization

Spec for letting approved vendors control the look of their **public** profile /
storefront page (`/vendors/:id`) from the **vendor portal**. Front-of-funnel planning
only — implementation flows through `/ship-card`. Related: [[Vendor & Admin Portals]] ·
[[Listing Types & Vendor Rules]] · [[Public Storefront & SSR]] · [[Category Taxonomy & Discovery]] ·
[[HB Domain Model]].

## Problem

The public vendor profile page shipped (PR #35, `/vendors/:id`) but it is **entirely
platform-controlled**: it renders `businessName` / `tradingName` / country + an
auto-derived list of category chips + the vendor's products in a flat grid. A vendor has
**zero say** in how their brand appears to customers. The vendor-portal "Profile" page is
still a `Coming soon` placeholder (`apps/web/src/app/features/vendor/pages/vendor-profile/vendor-profile.ts`).

Give vendors a **generic, brand-agnostic** way to shape that page: a **logo**, a **banner**
image, a short **slogan/tagline**, and the ability to **build their own product sections**
(e.g. "Our Top Picks"). Must work for every vendor with no per-vendor bespoke design.

## Scope (v1)

**In:**
- Vendor uploads a **logo** image and a **banner/hero** image from the portal.
- Vendor sets a short **slogan** (tagline) shown under the business name.
- Vendor **builds ordered product sections** on their profile. A section has a
  vendor-chosen **title** and one of two population types:
  - **curated** — an ordered, hand-picked set of the vendor's own products ("Our Top Picks").
  - **category** — auto-fills from one of the vendor's product categories, under a custom heading.
- The public `/vendors/:id` page renders all of the above, with graceful fallback to the
  current auto-derived behaviour when a vendor has set nothing.
- The vendor portal "Profile" page becomes a real editor (replaces the placeholder).

**Out (separate cards / TBD):**
- **`rule`-based auto sections** ("Newest N", "Best Sellers") — deferred to **VPC-6**.
  `newest` is cheap but "best sellers"/"top rated" need sales/rating signals that do not
  exist yet (see [[Listing Types & Vendor Rules]] backlog on vendor_rating / sales_velocity),
  so the whole `rule` type is pulled out of v1 and specced together in its own card.
- Free-form page-builder / drag-and-drop layout, custom colour themes, fonts, custom HTML.
  v1 is a fixed template with vendor-supplied content + product sections, not layout control.
- Surfacing logo/banner on the storefront "Featured SME Vendors" directory cards
  (`GET /vendors/directory`) — nice enhancement, defer.
- Moderation / approval workflow for vendor-uploaded imagery before it goes public
  (v1 trusts approved vendors — see open questions).
- CDN / object-storage for images (v1 reuses local disk `uploads/`, same as product images).

## Business rules it must honour

- **Owner-scoped writes only.** A vendor edits **only their own** profile. Ownership is
  already enforced in `VendorsService.update()` (admin OR `vendor.userId === currentUser.id`);
  the new upload/section endpoints MUST enforce the same. Never trust a client-supplied id
  for the write target. See [[Auth & Roles]].
- **Curated sections may only reference the vendor's OWN products.** Server-side guard:
  every `productId` in a curated section must belong to the acting vendor — otherwise a
  vendor could feature a competitor's product on their page. This is a real
  correctness/security requirement, enforced in the service layer.
- **Public shape stays lean.** Branding + sections are public-safe and may appear in
  `VendorResponseDto` (`GET /vendors/:id`). The public DTO must still **never** leak
  admin-only PII (`registrationNumber`, `verificationDocumentUrl`) — see [[Vendor & Admin Portals]].
- **Approved-only visibility unchanged.** `GET /vendors/:id` already returns approved
  vendors only; branding does not change who is visible.
- Sections reference **real** category / product ids; stale ids (deleted/unpublished
  products, empty categories) must degrade gracefully (skipped at render, never broken).
- Uploaded files follow the existing product-image guardrails: image MIME allow-list
  (`jpg|jpeg|png|webp`), size cap (products use 5 MB), disk storage, served from `/uploads`.

## What already exists (reuse — do NOT re-build)

- **Public profile page:** `apps/web/.../features/vendors/vendor-profile/` — fetches
  `VendorDto` via `VendorsService.getById`, lists products via `GET /products?vendorId=`,
  derives `categoryChips` client-side. This is the render target for the new fields, and
  it already loads the full vendor product list that curated sections resolve against.
- **Ownership-checked update:** `PATCH /vendors/:id` (`@Roles(VENDOR, ADMIN)`) →
  `VendorsService.update()` enforces owner-or-admin. `UpdateVendorRequest` /
  `UpdateVendorDto` already carry `businessName`, `tradingName`, `website`, `description`.
- **Image upload pattern:** `apps/api/src/products/upload/` — `multer.config.ts`
  (`diskStorage`, uuid filename, 5 MB, image MIME filter) + `FileUrlService.getFileUrl(filename, folder)`
  returning `/uploads/<folder>/<filename>`; `uploads/` served statically via
  `app.useStaticAssets` in `main.ts`. Copy this for a `uploads/vendors` folder.
- **`GET /vendors/me`** (`@Roles(VENDOR)`) → the portal's source for the signed-in vendor's
  own row. **Gap:** it currently returns the lean public `VendorResponseDto`, so the portal
  editor can't even read its own `website`/`description` today — the backend slice must widen
  the "me" view to include the vendor's own editable fields.

## `@hb/shared` contract impact

Interfaces + enums only; API DTOs `implement` them; web consumes them.

- `contracts/vendor.ts` — add public-safe fields to `VendorDto`:
  `logoUrl?: string`, `bannerUrl?: string`, `slogan?: string`,
  `profileSections?: VendorProfileSection[]` (order = array order).
- New `VendorProfileSection` interface (+ a `VendorSectionType = 'curated' | 'category'`
  union). Shape is designed so VPC-6 can add `'rule'` + `rule`/`limit` fields without a
  breaking change:
  ```ts
  interface VendorProfileSection {
    id: string;
    title: string;
    type: 'curated' | 'category';   // VPC-6 extends with 'rule'
    productIds?: string[];   // curated: ordered, hand-picked, vendor-owned
    categoryId?: string;     // category: auto-fills from the vendor's products
  }
  ```
- `UpdateVendorRequest` — add `slogan?: string` and `profileSections?: VendorProfileSection[]`
  (text/JSON set via `PATCH`; images come via dedicated multipart endpoints, not JSON).
- New owner-facing self view (e.g. `VendorSelfDto` or widen the `/vendors/me` response) so
  the portal editor can read its own `website`, `description`, `slogan`, `logoUrl`,
  `bannerUrl`, `profileSections`.
- Every new endpoint input = a class-validator DTO implementing the shared interface
  (nested `profileSections` validated with `@ValidateNested({ each: true })` + `@Type`;
  `slogan` length-capped; upload DTOs validate the file).

## Schema impact

New **nullable** columns on `vendors` (all nullable ⇒ clean migration, no backfill —
honours the "new column needs a default or is nullable" non-negotiable):

- `logoUrl varchar null`
- `bannerUrl varchar null`
- `slogan varchar null` (length-capped at the app layer)
- `profileSections jsonb null` (ordered array of `VendorProfileSection`)

One TypeORM migration, symmetric up/down. `synchronize` stays off. No money/inventory/order
state touched — but the update/upload service methods still get focused unit tests
(ownership + curated-products-belong-to-vendor guard + persistence + mapping).

## Design

Claude Design is the source of truth ([[Public Storefront & SSR]] / `docs/design/`). A
`vendor-profile` design bundle exists under `docs/design/`. New portal-editor (section
builder) and enhanced public-page visuals should be synced/added via `/design-sync` before/
with the frontend cards; build idiomatic Angular standalone components to the DESIGN.md
tokens — never paste exported markup.

## Resolved decisions

| Question | Decision |
|---|---|
| Category grouping vs custom sections? | **Vendor-built sections** (supersedes the earlier "featured categories" idea). A section = custom title + population type. |
| Section population types for v1? | **curated + category** only. `rule`-based auto sections (`newest`, `best-selling`) deferred to **VPC-6**. |
| Storage? | **JSONB** `profileSections` column on `vendors` — one column, one migration, read as a blob. Not a relational table for v1. |
| Predefined section names? | **No hardcoded names** (must stay generic per mission). Free-text title with suggested **preset chips** in the portal ("Top Picks", "New Arrivals") as a UX nudge only. |

## Open questions (flag for human)
1. **Image moderation** — v1 shows vendor-uploaded logo/banner publicly with no admin
   review (trusted approved vendors). Acceptable, or is a gate required? → *Recommend no
   gate for v1; revisit if abuse appears.*
2. **Caps** — RESOLVED via VPC-1: slogan ≤ 120 chars; ≤ 10 sections/vendor; ≤ 24 products per
   curated section. Confirmed with card owner.
3. **Directory cards** — should logo also show on the storefront "Featured SME Vendors"
   cards, or profile-only for v1? → *Recommend profile-only; directory is a follow-up.*

## Vertical slices (→ Trello cards)

1. **VPC-1 (backend):** Branding + sections contract + schema + text read/write.
   `@hb/shared` additions (branding fields + `VendorProfileSection` curated/category),
   migration (4 nullable columns incl. `profileSections jsonb`), expose on `VendorResponseDto`
   (`GET /vendors/:id`), widen `GET /vendors/me` to the owner self-view, extend
   `UpdateVendorDto` with `slogan` + validated `profileSections` (incl. the
   **curated-products-belong-to-vendor** guard + caps), unit tests. **Foundation — blocks 2/3/4/5.**
2. **VPC-2 (backend):** Logo & banner upload endpoints (`POST /vendors/me/logo` +
   `/banner`), reuse the products multer pattern into `uploads/vendors`. Depends on VPC-1.
3. **VPC-3 (frontend, portal):** Profile editor — text fields + slogan + logo/banner upload
   with preview. Depends on VPC-1/2.
4. **VPC-4 (frontend, portal):** **Section builder** — add/rename/reorder/delete sections;
   per type: curated product multi-picker (vendor's own products, ordered) or category
   picker; suggested title preset chips; persist `profileSections`. Depends on VPC-1.
5. **VPC-5 (frontend, public):** Render sections on `/vendors/:id` in order — resolve
   curated `productIds` and category sections against the fetched vendor products
   (skip dangling ids), with graceful fallback to today's auto-derived behaviour when unset.
   Depends on VPC-1 (best after 3/4).
6. **VPC-6 (DEFERRED — backlog):** **Rule-based auto sections.** Extend `VendorProfileSection`
   with `type: 'rule'` + `rule` (`newest` | `best_selling`) + `limit`; portal rule editor;
   public render. **`best_selling` needs a sales-signal data model first** (no
   sales/rating data exists today — this card must spec that prerequisite or split it out).
   `newest` is trivially derivable and could ship first. Not part of the v1 batch.

## Handoff

These slices become Trello cards in **To Do**. `/ship-card <card-id>` pulls them.
VPC-1 is the keystone (contract + schema + section validation); everything else builds on it.
VPC-6 is a backlog marker — do not pick up until a human prioritizes it.


## Implementation Notes

**VPC-1 shipped** (Trello ZpvX9XIv, July 2026). Backend keystone for vendor branding + profile-sections contract, schema, and text read/write. All tests/lint/build green; PR opened; moved to "In Review".

### What shipped

**`@hb/shared` contract widening (`libs/shared/src/contracts/vendor.ts`):**
- `VendorDto` extended with `logoUrl?: string`, `bannerUrl?: string`, `slogan?: string`, `profileSections?: VendorProfileSection[]`.
- New `VendorProfileSection` interface: `{ id: string, title: string, type: 'curated' | 'category', productIds?: string[], categoryId?: string }`.
- Promoted `VendorSectionType` to a real enum in `libs/shared/src/enums/vendor-section-type.ts` with values `CURATED` / `CATEGORY`, left open for future `RULE` member per design.
- New `VendorSelfDto` (owner self-view): extends `VendorDto` + adds `website` and `description` fields, solving the spec's noted gap that `GET /vendors/me` could not read the vendor's own editable fields.
- `UpdateVendorRequest` gained `slogan?: string` + `profileSections?: VendorProfileSection[]`.
- New `VendorProfileSectionDto`, `VendorSelfResponseDto` DTOs in API layer.
- Existing `VendorResponseDto` / `AdminVendorResponseDto` map the new branding fields; public shape still never exposes `registrationNumber` / `verificationDocumentUrl`.
- `findByUserId()` and `GET /vendors/me` now return the widened self-view (previously returned the lean public DTO).
- `VendorsService.update()` also now returns the self-view shape for consistency.

**Schema migration** (`1784332800000-VendorBrandingSections`):
- 4 nullable columns on `vendors` table: `logoUrl varchar null`, `bannerUrl varchar null`, `slogan varchar null`, `profileSections jsonb null`.
- Symmetric up/down; no backfill needed.

**Caps confirmed** (resolved open question #2):
- Slogan: ≤ 120 characters.
- Profile sections: ≤ 10 per vendor.
- Curated section product count: ≤ 24 `productIds` per section.

**Security-critical guard shipped** (`VendorsService.update()`):
- Every `productId` in a curated section must belong to the vendor row being edited (batched single query, `BadRequestException` on mismatch). Prevents one vendor from featuring a competitor's product on their page.
- Review round 1 caught a real bypass: a `category`-typed section could otherwise carry unvalidated `productIds` past the guard.
- Fixed via two custom class-validator `registerDecorator` validators (`IsCuratedProductIds`, `IsCategoryId`) that structurally enforce the discriminated union:
  - Curated requires 1–24 v4-uuid `productIds`, forbids `categoryId`.
  - Category requires `categoryId`, forbids `productIds`.
- Service-level duplicate-section-id and duplicate-productId-in-section guard also added.

**Test coverage:**
- Unit tests for curated-section persistence, cross-vendor productId rejection (including category-type bypass path, tested end-to-end), admin-editing-another-vendor's-profile, duplicate section-id/productId rejection.
- Self-view shape verified; new `vendor-profile-section.dto.spec.ts` with boundary tests (64/65 char section-id cap, 120/121 slogan cap, 24/25 productId cap) and cross-field validation tests.

### Out of scope (by design)

- Logo/banner have no write path yet. Images arrive via VPC-2's multipart upload endpoints, not this JSON PATCH. Schema/contract only.
- No UI work in VPC-1 (portal editor built in VPC-3/4).
- No `categoryId` existence validation (dangling category ids degrade gracefully at render time per spec's stale-id handling rule). That's VPC-5's concern.

### Review outcome

- **Round 1:** FIX-FIRST. Five issues found and fixed: the security bypass noted above, four correctness/coverage gaps (test edge-cases on caps, confirm self-view shape consistency, verify duplicate guards, test admin-edit-other-vendor path).
- **Round 2:** SHIP. All findings addressed; tests green.

### PR + card status

PR link: [see Trello card comment / link — lead will fill in].
Card moved to "In Review" pending human merge approval.

### Follow-ups (non-blocking)

- VPC-2..5 now unblocked.
- Stale `profileSections` ids not re-validated at read time (degrade gracefully); candidate for future validation pass if UX signals a need.
- `vendorId` could theoretically be undefined in ownership-check query filter (though currently unreachable code path); minor defensive guard candidate.
- Some DTO duplication (`VendorResponseDto` + `VendorSelfResponseDto` + admin variant) noted; candidate for a future refactor pass, not urgent.


**VPC-2 shipped** (Trello AAjMPwV7, July 2026). Backend upload endpoints for vendor logo and banner images. All tests/lint/build green; PR opened; moved to "In Review".

### What shipped

**Two new multipart upload endpoints on `VendorsController` (`apps/api/src/vendors/vendors.controller.ts`):**
- `POST /vendors/me/logo` — `@Roles(UserRole.VENDOR)`, `FileInterceptor('file', vendorImageMulterOptions)`, owner-scoped strictly via `@GetUser()` (never a client-supplied id) — same ownership idiom as `GET /vendors/me`.
- `POST /vendors/me/banner` — identical guard and pattern.

**New upload infrastructure (`apps/api/src/vendors/upload/`):**
- `vendor-image.multer.config.ts` — Multer disk storage into `uploads/vendors`, 5MB cap, mimetype filter (`jpg|jpeg|png|webp`), extension derived from a fixed mimetype→extension allow-list (not from client-supplied `originalname`). Mirrors the existing `products/upload/` pattern but fixes a critical extension-injection bug (see security note below).
- `vendor-image-file.pipe.ts` — shared `ParseFilePipeBuilder` pipe with `FileTypeValidator` configured for disk-stored uploads via `fallbackToMimetype: true` (required; NestJS magic-number validation reads `file.buffer` which is empty under `diskStorage` — without the fallback, all uploads were silently rejected after Multer had already written the file).

**Service-layer image write (`VendorsService.updateLogo` / `updateBanner`):**
- Private `updateBrandingImage(userId, field, file)` — finds vendor by `userId` (404 if missing), retrieves file URL via `FileUrlService.getFileUrl(filename, 'vendors')`, persists URL to the vendor row (updates `logoUrl` or `bannerUrl` column), returns `VendorSelfResponseDto` for response-shape consistency with `GET /vendors/me`.
- `FileUrlService` (reused from products module) registered as an additional provider in `VendorsModule` — not duplicated.
- No schema or contract changes — `logoUrl`/`bannerUrl` columns and the self-view DTO already existed from VPC-1.

**Two critical security bugs discovered during review and fixed in this PR (reusable lessons):**

1. **NestJS `FileTypeValidator` + diskStorage silent rejection bug:** The validator does magic-number validation by reading `file.buffer`. Under Multer's `diskStorage` (not `memoryStorage`), `file.buffer` is never populated — only `file.filename` is set. Without `fallbackToMimetype: true` on `ParseFilePipeBuilder`, every disk-stored file was silently rejected with a false 422, *after* Multer had already written the file to disk. **Fixed:** added `fallbackToMimetype: true` to the pipe config, verified against the installed NestJS source, and added regression tests (`vendor-image-file.pipe.spec.ts`).

2. **Extension-injection vulnerability in Multer filename callback:** The pre-existing `products/upload/multer.config.ts` (and initially replicated in VPC-2's config) derived the stored file's extension from the client-supplied `originalname`, gated only by a `mimetype` regex check. Since `mimetype` is client-supplied and `uploads/` is served statically from the API origin (same origin as the httpOnly refresh cookie), a crafted request with `allowedMimetype + originalname: 'x.html'` could persist and later serve back as `text/html` from the API origin, enabling a stored XSS attack against the refresh token. **Fixed:** VPC-2's config now derives the extension from a fixed mimetype→extension allow-list instead of `extname(originalname)`. **Follow-up:** the identical bug exists in the pre-existing `products/upload/multer.config.ts` (flagged separately, not fixed in this PR — out of scope).

### Out of scope (by design)

- No portal image-upload UI widget (built in VPC-3).
- No orphan-file cleanup on image replace (old logo/banner not unlinked from disk) — matches existing products-upload behaviour, consistent rather than a regression, left as-is.
- Frontend sections (VPC-3/4/5) untouched.

### Review outcome

- **Round 1:** FIX-FIRST. Two critical security/correctness issues found and fixed: the `fallbackToMimetype` silent rejection bug, and the extension-injection vulnerability. Plus coverage: ensure regression tests prevent reverts, confirm both bugs against installed source.
- **Round 2:** SHIP. Both fixes verified; new tests for the regression cases added (`vendor-image-file.pipe.spec.ts`, extended `vendor-image.multer.config.spec.ts`). Full vendors test suite: 5 suites / 95 tests green. `npm run lint:api` clean. `npm run build` clean. One pre-existing, unrelated test-suite failure noted (`search/search-visibility.integration.spec.ts`, a DI resolution issue) — confirmed independently of this branch via `git stash`, flagged separately, not touched here.

### PR + card status

PR link: [see Trello card comment / link — lead will fill in].
Card moved to "In Review" pending human merge approval.

### Follow-ups (non-blocking)

- VPC-3 (portal profile editor with upload widget) now unblocked.
- **Products-upload extension bug:** the identical extension-injection vulnerability exists in `apps/api/src/products/upload/multer.config.ts` (pre-existing, discovered during VPC-2 review). Flagged as a separate follow-up task — medium priority, same fix pattern.
- Minor advisory notes from review (collapsing `productImageMulterOptions` / `vendorImageMulterOptions` into a shared helper, exporting `FileUrlService` from a shared module) — not urgent, candidate for future refactor.


**VPC-3 shipped** (Trello y3fz4TC2, July 2026). Frontend portal editor for vendor profile — text fields + image uploads. All tests/lint/build green; PR opened; moved to "In Review".

### What shipped

**Vendor profile portal editor** (`apps/web/src/app/features/vendor/pages/vendor-profile/`):
- Replaced `Coming soon` placeholder with a real reactive form component; follows the sibling `vendor-products` portal-page pattern.
- `VendorsService` (`apps/web/src/app/core/api/vendors.service.ts`): `getMe()` widened to return `VendorSelfDto` (was `VendorDto`); `update()` widened to return `VendorSelfDto` (server already returns the self-view from `PATCH /vendors/:id`, trusting response avoids re-deriving state client-side). New `uploadLogo(file)` / `uploadBanner(file)` multipart POST to `/vendors/me/logo` and `/vendors/me/banner` (field name `file`).
- **`VendorProfile` component** (`.ts`/`.html`/`.scss`):
  - Typed reactive form: `businessName` (required), `tradingName`, `website`, `description`, `slogan` (client `maxLength(120)` mirrors VPC-1 server cap).
  - Save via `PATCH /vendors/:id`; pending/success/error signal state + no-double-submit guard.
  - Blank optional fields normalised to `undefined` before PATCH (not sent as empty strings).
  - **Logo/banner upload unified handler** (`private uploadImage(kind, event)`): file-input change → client-side 5MB size check → local `URL.createObjectURL` preview (SSR-guarded via `isPlatformBrowser`) → multipart upload → on success, server response replaces vendor state (object URL revoked); on error, rejected blob preview discarded, falling back to last-persisted image (bug fix — see review outcome below).
  - Styled to `docs/design/DESIGN.md` tokens; reuses existing portal-shell visual language (matches `admin-catalog` / `vendor-products`). No new Claude Design export needed.
  - Out of scope by design: `profileSections` (section builder) — that is VPC-4.

### Key decisions

- **No MatSnackBar** — matches existing codebase convention (no Material snackbar usage); kept `.error-banner` / `.success-banner` plain-div pattern used by `vendor-products`, `profile-addresses`, etc.
- **Immediate upload on file selection** — instant local preview, swap to server URL on success; single filepicker (no separate upload button). Low-friction pattern.
- **Trust server response for state** — avoid manual field-merging; widened service to return `VendorSelfDto` ensures form state tracks the authoritative server state post-save.
- **Unified logo/banner handler** — one `uploadImage(kind, event)` method handles both paths (previously duplicated). Reduces maintenance burden.

### Review outcome

- **Round 1:** FIX-FIRST. Two real bugs found and fixed in a follow-up commit:
  1. **Upload error left blob preview rendered:** Failed logo/banner upload left the rejected local blob preview rendered as live (and leaked the object URL). Error handler did not reset the preview signal the way success handler did. **Fixed:** reset and revoke preview in both success and error branches.
  2. **Form state echoed local values, not server response:** Save handler propagated local raw form values (`website`, `description`) back into component state after a successful PATCH instead of trusting the server response. Happened because it was typed against generic `VendorDto` rather than self-view. **Fixed:** widened `VendorsService.update()` to return `VendorSelfDto` and use the response directly; deleted ~10 lines of manual field-merging.
  3. (Also addressed in the same pass: unified logo/banner upload handlers, added client-side 5MB pre-check, normalised blank optional fields to `undefined`, cleared stale success/error banners when user resumes editing after a save.)
- **Round 2:** SHIP. All findings addressed; tests/lint/build green.

### Test coverage

- `VendorProfile` component spec (`vendor-profile.spec.ts`): form load/populate, save payload shape (incl. blank-field omission), save success trusts server response, save error, required-field guard, double-submit guard, slogan `maxLength` boundary, stale-banner-clears-on-edit, logo/banner upload success + preview lifecycle + error-resets-to-persisted-image, client-side size-limit rejection, load-error path.
- `VendorsService` spec additions (`vendors.service.spec.ts`): `getMe()` self-view shape, `update()` self-view shape, `uploadLogo()` / `uploadBanner()` multipart request shape.
- Full suite: **638 tests / 55 files green**. `npm run build` clean (pre-existing unrelated SCSS budget warning on `admin-catalog`, not touched). `npm run lint:api` clean.

### PR + card status

PR link: [see Trello card comment / link — lead will fill in].
Card moved to "In Review" pending human merge approval.

### Follow-ups (non-blocking)

- None new. VPC-4 (section builder) and VPC-5 (public render) remain the next slices in the batch, both already unblocked since VPC-1 shipped.


**VPC-4 shipped** (Trello d7IHQ8Rm, July 2026). Frontend section builder for vendor profile — add/rename/reorder/delete sections; curated product picker and category picker. All tests/lint/build green; PR opened; moved to "In Review".

### What shipped

**"Product sections" panel added to the vendor-profile portal page** (`apps/web/src/app/features/vendor/pages/vendor-profile/vendor-profile.{ts,html,scss}`):
- Reuses the established vendor-portal-shell visual pattern (no new Claude Design export needed); same page VPC-3 built.
- **Section CRUD:**
  - Add: inline form to select type (curated / category) + free-text title; preset-chip UX nudges (non-forcing "Top Picks"/"New Arrivals" chips that auto-fill title for convenience).
  - Rename: inline edit, 120-char clamp mirroring server `@MaxLength(120)` from VPC-1.
  - Reorder: up/down buttons (keyboard-operable, no drag-only widget) — one mental model reused at both section-level and product-level.
  - Delete: remove from array.
- **Curated sections:** ordered multi-picker strictly scoped to the vendor's own products (`productsService.list({ vendorId, limit: 100 })`), with up/down reorder within the section. Soft 24-products-per-section UX cap (server remains source of truth).
- **Category sections:** single-select picker over categories derived by deduping the vendor's own fetched products' categories (not the full platform catalog).
- **Section `id` generation:** client-side via `crypto.randomUUID()` — safe under SSR because it only runs from a click handler, never during server render.
- **Save distinct action:** `saveSections()` separate from the existing business-details "Save changes"; PATCHes the full `profileSections` array via `VendorsService.update()` and trusts the server response back into state (same convention VPC-3 established). Empty `[]` is a valid, savable state.
- **Client-side validation gate** (added in review round 1): Save button disables and per-row hints appear when a curated section has 0 products, a category section has no categoryId, or a title is blank — preventing guaranteed server 400s that would reject the *entire* PATCH including valid sibling sections.
- **Truncation warning:** visible "showing first 100 products" warning when vendor has more products than the fetch returns (`total > items.length`), rather than silently hiding already-picked products outside that page.

### Key decisions

- **Up/down buttons (not drag-and-drop)** — one mental model reused at both section-level and product-level, fully keyboard-operable, no new dependency.
- **Reorder by product id, not list index** — keyed off product id via `indexOf` into the raw `productIds` array (not the filtered/resolved product list). This distinction prevented a desync bug during review (see below).
- **Trust server response** — follow VPC-3's convention; no client-side field-merging after save.

### Review outcome

- **Round 1:** FIX-FIRST. Four blocking issues found and fixed:
  1. **UI allowed structurally invalid sections:** curated with 0 products or category with no categoryId would PATCH and get a server 400, failing the *entire* request including valid sibling sections. **Fixed:** added `sectionsValidForSave` guard disabling Save button + per-row hints for each invalidity cause (0 products, no categoryId, blank title).
  2. **Reorder button index desync:** reorder operated in the *filtered* resolved-product index space while mutating the *raw* `productIds` array. A section containing one unresolvable/stale product id would desync, silently reordering the wrong entry. **Fixed:** keyed reorder off product id via `indexOf` into the raw array instead of list position.
  3. **Silent truncation in product picker:** 100-product fetch page truncates for high-catalog vendors; already-picked ids outside that page vanished from view while still riding in the save payload. **Fixed:** visible "showing first 100 products" warning when `total > items.length`.
  4. **No trim/length guard on inline rename:** server allows blank, caps at 120 chars; HTML `maxlength` attribute was the only gate. **Fixed:** added `.slice(0,120)` trim in `renameSection()` method (folded blank titles into the same validity guard as issue 1).
- **Round 2:** SHIP. All four findings verified genuinely closed (re-derived each original failure scenario against the fix, not just re-read the diff).

### Test coverage

- ~30+ new/updated Vitest cases in `vendor-profile.spec.ts`: add/rename/reorder/delete round-trips (including that reorder and rename land in the *saved payload*, not just local state), curated picker strictly scoped to vendor's own products, category options strictly scoped to categories the vendor has products in, preset-chip prefill (non-forcing), curated/category save payload shape (incl. discriminated union correctness), empty-sections save, trust-server-response convention, save-blocked-until-valid for all three invalidity causes, reorder-by-id correctness under a simulated stale/unresolvable-id desync scenario, 10-section and 24-product soft caps disabling their respective add controls, 120-char rename clamp.
- Full suite: **671 tests / 55 files green** (`npm run test -w @hb/web`). `npm run build` clean (one pre-existing SCSS budget warning on `admin-catalog`, not touched). `npm run lint:api` clean (no backend touched).

### Out of scope (by design)

- No `rule`-based sections (`newest` / `best_selling`) — that is VPC-6, explicitly deferred.
- No rendering of sections on the public profile page — that is VPC-5, the next slice.
- Full pagination/search in the product picker beyond the visible truncation warning (accepted follow-up).

### PR + card status

PR link: [PR link — lead filled in after doc pass].
Card moved to "In Review" pending human merge approval.

### Follow-ups (non-blocking)

- `addSection()` should mirror `renameSection()`'s defensive `.slice(0,120)` title clamp in TS (currently relies only on the HTML `maxlength` attribute).
- Interpolate the "first 100 products" warning copy against the `PRODUCT_LIST_MAX` constant instead of a hardcoded literal.
- The caps `10` (sections) and `24` (products/section) now live in three places (`vendor-profile.ts`, `update-vendor.dto.ts`, `vendor-profile-section.dto.ts`) — candidate for promoting to `@hb/shared` constants to remove drift risk (flagged by review as advisory, not blocking).
- Product picker full pagination/search for vendors with >100 products (accepted follow-up, not blocking for v1).
- VPC-5 (public render of sections) is now the natural next slice.


**VPC-5 shipped** (Trello rHxbUA2G, July 2026). Frontend-only slice rendering vendor branding and profile sections on the public profile page. All tests/lint/build green; PR not yet opened (lead opens after doc pass); card moves to "In Review" after PR opens.

### What shipped

**Frontend-only slice** (no `@hb/shared`/backend changes; the contract shipped in VPC-1). Changed files: `apps/web/src/app/features/vendors/vendor-profile/vendor-profile.ts`, `.html`, `.scss`, `.spec.ts`.

- **Two new computed signals** on `PublicVendorProfile` component:
  - `resolvedSections` — resolves `vendor.profileSections` in array order against the already-loaded `products()` signal (no extra HTTP requests). Curated sections map `productIds` in given order to fetched products, dropping dangling ids; category sections filter fetched products by `categories[].id === categoryId`; a section resolving to zero products is dropped entirely rather than rendering an empty shell; returns `[]` while `productsState() === 'loading'` to avoid a flash.
  - `hasCustomSections` — true iff at least one section resolved non-empty.
- **Hero enhancements:** `bannerUrl` renders a wide hero image above the hero card (`loading="eager" fetchpriority="high"` — above-the-fold LCP element); `logoUrl` renders a circular logo next to the business name; `slogan` renders under the trading name. All optional/guarded, plain SSR-safe `<img>` tags, no browser-only APIs.
- **Conditional layout:** when `hasCustomSections()` is true, the page renders each resolved section (heading + product grid) instead of the old auto-derived category-chips + flat-grid. When false (vendor set nothing, or every section resolved empty), the original auto-derived layout renders unchanged — verified byte-identical fallback, no regression.
- **Residual grid:** a **"More from `<vendor businessName>`"** grid renders every fetched product not claimed by any resolved section, positioned after the vendor's own sections. A product referenced by more than one section renders in every section that claims it and is excluded from the residual grid as long as at least one section claims it. (Prevents silent content loss if a vendor builds even one small curated section.)
- No new Claude Design export pulled for this — built to the existing `docs/design/DESIGN.md` token language and the existing page's visual rhythm.

### Key decisions

**Residual-grid approach** (made mid-review, confirmed with the card owner): Without a residual grid showing unclaimed products, a vendor building even one small curated section would make every other product on their page invisible — a silent content-loss regression. The alternative considered was "sections-only, full curation control" (unsectioned products stay findable via search, just not on the vendor's own page) — the residual-grid option was chosen instead to maximize discoverability for vendors transitioning to sections.

### Known limitation (inherited, not a regression)

Curated/category resolution only operates against the vendor's already-fetched product list, capped at 100 products (`PRODUCT_LIST_MAX`, pre-existing since PR #35, untouched by this card). A vendor with >100 products who curates a product past that fetch boundary would see it silently absent. This limitation is documented separately (see VPC-4 follow-ups: "Product picker full pagination/search for vendors with >100 products"). Not a new regression — the residual grid means the worst case is now no worse than before this card (content loss is bounded by the fetch, not by section resolution).

### Review outcome

- **Round 1:** FIX-FIRST. Two blocking findings:
  1. **Silent-content-loss issue:** the 100-product-cap concern was investigated and confirmed as a pre-existing, already-documented limitation (not a regression introduced by this card), left as an accepted follow-up rather than fixed in this card.
  2. **Type safety & accessibility:** explicit `VendorSectionType.CATEGORY` type check (was an implicit `else`, now future-proofed against a later third section type per the enum's own "left open" contract comment); `aria-labelledby` added pointing at each section's own heading `id` (replaced a redundant `aria-label` duplicating the visible `<h2>`); `loading="eager" fetchpriority="high"` added to the banner image (was `loading="lazy"`, wrong for an above-the-fold LCP element).
- **Round 2:** SHIP. Both round-1 blockers verified genuinely closed (re-derived the original failure scenarios against the fix, including the multi-section-overlap case).

### Test coverage

`vendor-profile.spec.ts` — new `describe('vendor branding + profile sections', ...)` block: hero banner/logo/slogan present/absent; curated-section resolution preserves `productIds` order (not fetch order); dangling ids silently dropped; category-section filtering by `categoryId`; all-sections-empty falls back to the original chips+grid layout; fallback also applies when `profileSections` is `undefined` or `[]`; multiple sections render in given array order; no section markup renders while products are still loading; residual grid renders unclaimed products with the correct heading; residual grid is omitted when every product is claimed; a product referenced by two sections renders in both and is excluded from the residual grid.

Full web suite: **684 tests / 55 files green** (`npm run test -w @hb/web`). `npm run build` clean (two pre-existing unrelated SCSS budget warnings: `admin-catalog` and the vendor-portal's `vendor-profile.scss` — a different file from this diff). `npm run lint:api` clean (no backend touched).

### Out of scope (by design)

- No `rule`-based sections (VPC-6, explicitly deferred).
- No fix to the pre-existing 100-product fetch cap (documented limitation, accepted follow-up).
- No portal changes (VPC-3/4 already shipped the editor).

### PR + card status

PR not yet opened at time of documentation (lead opens it right after this doc pass). Card moves to "In Review" after PR opens.

### Follow-ups (non-blocking)

- Section-id character-set validation (`@Matches` on the DTO) to harden the `aria-labelledby` wiring — section id (vendor-controlled free text, `@MaxLength(64)`, no character-set restriction) is interpolated directly into a DOM `id`/`aria-labelledby`. A section id containing a space or other special characters could theoretically degrade accessibility wiring (low severity, not reachable through the portal's own UI which generates ids via `crypto.randomUUID()`).
- Logo could switch off `loading="lazy"` since it's above the fold; currently lazy-loaded, not the LCP element so minor.
- The 100-product fetch cap should eventually get real pagination (tracked informally, not a card yet) — this note now documents that limitation centrally for a future prioritization pass, since previously it was only mentioned in VPC-4's follow-ups as a portal-picker-specific issue.
