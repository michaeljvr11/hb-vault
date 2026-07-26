# Session 18 — VPC-2 Documentation Pass

**Date:** July 26, 2026  
**Card:** VPC-2 (Trello AAjMPwV7, "Vendor logo & banner image upload endpoints")  
**Status:** SHIPPED — documented, PR ready

## What shipped

Backend upload endpoints for vendor branding images.

### Endpoints & infrastructure

- `POST /vendors/me/logo` and `POST /vendors/me/banner` on `VendorsController` — `@Roles(UserRole.VENDOR)`, owner-scoped via `@GetUser()` (never client-supplied id).
- New `apps/api/src/vendors/upload/vendor-image.multer.config.ts` — disk storage, 5MB cap, mimetype filter, **extension from fixed allow-list** (not from `originalname`).
- New `apps/api/src/vendors/upload/vendor-image-file.pipe.ts` — `ParseFilePipeBuilder` with `fallbackToMimetype: true`.
- `VendorsService.updateLogo()` / `updateBanner()` backed by private `updateBrandingImage(userId, field, file)` — returns `VendorSelfResponseDto` for response-shape consistency.
- `FileUrlService` reused from products module (not duplicated).
- No new schema or contract — `logoUrl`/`bannerUrl` and self-view DTO existed from VPC-1.

### Critical bugs found & fixed

1. **FileTypeValidator + diskStorage silent rejection:** NestJS reads `file.buffer` for magic-number validation; diskStorage never populates it. Without `fallbackToMimetype: true`, all disk-stored uploads were false-rejected 422 *after* Multer wrote the file. Fixed; regression tests added.

2. **Extension-injection XSS in Multer filename callback:** Pre-existing in `products/upload/multer.config.ts`, replicated initially in VPC-2's config — derived extension from client-supplied `originalname`, gated only by `mimetype` regex. Allows crafted `originalname: 'x.html'` + allowed mimetype to serve back as `text/html` from the API origin (same-origin XSS against httpOnly refresh cookie). Fixed in VPC-2 by using fixed mimetype→extension allow-list. Flagged separately for products upload.

## Key decisions

- **fallbackToMimetype: true** — required for disk-stored file validation; magic-number read only works on memoryStorage.
- **Mimetype-based extension allow-list** — replaces `extname(originalname)` to prevent extension injection; extension must match the mimetype.
- **FileUrlService in VendorsModule** — reused from products, not duplicated; keeps upload pattern consistent.
- **VendorSelfResponseDto response shape** — matches `GET /vendors/me` for consistency.

## Test & review outcome

- **Round 1:** FIX-FIRST — both security issues flagged, coverage gaps noted.
- **Round 2:** SHIP — both fixes verified against installed source, regression tests added. Vendors suite: 5 suites / 95 tests green. Lint/build clean. One pre-existing unrelated test failure noted (search-visibility DI issue, independently confirmed, not touched).

## Artifacts updated

- `Vendor Profile Customization.md` (Obsidian) — appended VPC-2 subsection to Implementation Notes.

## Follow-ups spawned

- **Products-upload extension bug** — identical vulnerability in `apps/api/src/products/upload/multer.config.ts`, flagged as separate task (medium priority, same fix pattern).
- **Search-visibility DI fix** — pre-existing unrelated test failure, flagged separately.
- Minor DTO/provider duplication advisory (future refactor, not urgent).

## Next steps

- Lead opens PR (link to be filled in Trello card comment).
- VPC-3 (portal profile editor) now unblocked.
