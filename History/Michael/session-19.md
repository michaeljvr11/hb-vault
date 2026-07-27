# Session 19 — VPC-3 Documentation Pass

**Date:** July 27, 2026  
**Card:** VPC-3 (Trello y3fz4TC2, "Vendor profile portal editor — text + image uploads")  
**Status:** SHIPPED — documented, PR ready

## What shipped

Frontend portal editor for vendor profile customization — replaces the `Coming soon` placeholder with a real reactive form and integrated logo/banner upload widgets.

### Portal editor & service widening

- `VendorsService` (`apps/web/src/app/core/api/vendors.service.ts`): `getMe()` and `update()` both widened to return `VendorSelfDto` (was `VendorDto`), enabling portal to read and persist the vendor's own editable fields; new `uploadLogo()` / `uploadBanner()` multipart POST methods to `/vendors/me/{logo,banner}` (field name `file`).
- **`VendorProfile` component** (`apps/web/src/app/features/vendor/pages/vendor-profile/`):
  - Typed reactive form: `businessName` (required), `tradingName`, `website`, `description`, `slogan` (client `maxLength(120)` mirrors server cap from VPC-1).
  - Save via `PATCH /vendors/:id`; pending/success/error signal state + no-double-submit guard; blank optional fields normalised to `undefined` (not sent as empty strings).
  - **Unified logo/banner upload handler** (`private uploadImage(kind, event)`): file-input change → client-side 5MB size check → local `URL.createObjectURL` preview (SSR-guarded) → multipart upload → success swaps to server URL (revokes object URL); error discards rejected preview and falls back to persisted image.
  - Styled to `docs/design/DESIGN.md` tokens; reuses existing portal-shell language (matches `admin-catalog` / `vendor-products`). No new Claude Design export.

### Key design choices

- **No MatSnackBar** — follows codebase convention; kept `.error-banner` / `.success-banner` plain-div pattern from `vendor-products`, `profile-addresses`.
- **Immediate upload on file selection** — single filepicker, no separate upload button; instant local preview swaps to server URL on success. Low-friction UX.
- **Service returns `VendorSelfDto`** — avoids manual field-merging client-side; form state always tracks authoritative server response post-save.
- **Unified upload handler** — one method for logo and banner (previously duplicated logic).

## Review outcome

- **Round 1:** FIX-FIRST — two real bugs caught:
  1. **Upload error left blob preview rendered:** Failed upload didn't reset the preview signal in the error branch (only success branch did); leaked object URL. Fixed by resetting/revoking in both branches.
  2. **Form state echoed local values, not server response:** Save handler propagated raw local form values back into state instead of trusting server response (was typed against generic `VendorDto`, not self-view). Fixed by widening service to return `VendorSelfDto` and using response directly (~10 lines of field-merging deleted).
  - Also addressed: unified upload handlers, added client-side 5MB pre-check, normalised blank optional fields, cleared stale banners on edit resume.
- **Round 2:** SHIP — all findings addressed. Tests/lint/build green.

## Test & artifacts

- `vendor-profile.spec.ts`: form load/populate, save shape (incl. blank-field omission), success trusts response, error handling, required-field guard, double-submit guard, slogan boundary, stale-banner-clear, upload success + preview lifecycle + error-reset-to-persisted, size-limit rejection, load-error.
- `vendors.service.spec.ts`: `getMe()` self-view shape, `update()` self-view shape, multipart upload request shapes.
- **Full suite: 638 tests / 55 files green.** `npm run build` clean (pre-existing SCSS budget warning on `admin-catalog`, not touched). `npm run lint:api` clean.

## Artifacts updated

- `Vendor Profile Customization.md` (Obsidian) — appended VPC-3 subsection to Implementation Notes.

## Follow-ups spawned

- None new. VPC-4 (section builder) and VPC-5 (public render) remain unblocked and next in batch.

## Next steps

- Lead opens PR (link to be filled in Trello card comment).
- VPC-4 can now begin (section builder UI — add/reorder/delete sections, curated product multi-picker, category picker).
