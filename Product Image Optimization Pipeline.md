# Product Image Optimization Pipeline

**Status:** Spec — not yet sliced onto the board (see "Handoff gap" at the bottom).
**Date:** 2026-08-16
**Related:** [[HB Domain Model]] · [[Vendor & Admin Portals]] · [[Vendor Profile Customization]] · [[Landing Site Migration]] · [[Public Storefront & SSR]] · [[Product Search Engine]] · [[Listing Types & Vendor Rules]] · [[Auth & Roles]]

## Problem

Vendor- and admin-uploaded product images are stored and served **byte-for-byte as
uploaded** — original resolution, original format, up to 5 MB each, up to 8 per product.
A vendor photographing stock on a phone can put eight 4000×3000 JPEGs (≈40 MB) behind one
product page, and every storefront grid card, PDP hero and search result then downloads
the full-resolution original to render it at 200–600 px.

This is a direct hit on the audience this platform exists for: [[Landing Site Migration]]
already flagged that unoptimised imagery "undercuts the point of prerendering for a
mobile-data audience" — Namibian shoppers on metered mobile data. That note fixed it for
five *static brand assets*; nothing protects the *uploaded* catalogue, which is unbounded
and grows with every vendor.

## Confirmed current state (code-verified 2026-08-16 — do not re-derive)

| Area | Reality |
|---|---|
| Upload entry point | `POST /products`, `FilesInterceptor('images', 8, productImageMulterOptions)` — `apps/api/src/products/products.controller.ts` |
| Multer config | `apps/api/src/products/upload/multer.config.ts` — `diskStorage` → `./uploads/products`, filename `uuid + ext`, 5 MB cap, mimetype allow-list `jpg\|jpeg\|png\|webp` |
| Second validation layer | `ParseFilePipeBuilder` in the controller repeats the *same* two checks (file type + 5 MB) |
| Dimension validation | **None.** Nowhere in the codebase. |
| Server-side processing | **None.** No `sharp` / `jimp` / `imagemin` in any `package.json`. |
| Serving | `app.useStaticAssets(join(process.cwd(), 'uploads'), { prefix: '/uploads' })` — `apps/api/src/main.ts:61` |
| URL mapping | `FileUrlService.getFileUrl(filename, folder='products')` → `/uploads/<folder>/<filename>` |
| Persistence | `ProductsService.createWithImages` maps each file → `CreateProductImageDto { url, key, isPrimary, displayOrder, altText }` |
| Entity | `ProductImage` (`apps/api/src/products/entities/product-image.entity.ts`) — `url`, `key`, `isPrimary`, `displayOrder`, `altText`. No dimensions, no size, no variants. |
| Shared contract | `ProductImageDto` (`libs/shared/src/contracts/product.ts:14-20`) — `id`, `url`, `isPrimary`, `displayOrder`, `altText?`. Same gap. |
| Web consumption | `product-card.ts` `primaryImage()` → single `<img [src]>` with `loading="lazy"`; `product-detail.html:33` → single `<img [src]="img.url">`. No `srcset`, no `<picture>`, no `NgOptimizedImage` anywhere in `apps/web`. |
| Search index | `search-document.ts:43,64` stores `imageUrl` = primary image's `url` — the raw original is what search results render too. |
| Image mutation | `PATCH /products/:id` does **not** touch images (documented in [[Vendor & Admin Portals]]); images exist only on create. |
| Disk cleanup | None — replaced/deleted images are never unlinked. Pre-existing, accepted, documented in [[Vendor Profile Customization]]. |

## Prior art — what this is NOT building on

Commit `a2e7242` ("perf(web): optimize the LSM brand assets and serve heroes responsively")
is **not** a reusable pipeline. It was a one-off build-time script run once against five
static files in `apps/web/public/`, emitting WebP + JPEG fallbacks at fixed breakpoints
consumed via `<picture>`/srcset. Useful as a **precedent for the output shape and the
quality/format choices**, and as proof the team accepts WebP-with-fallback. It gives
uploads nothing to route through. See [[Landing Site Migration]].

## Conflicts & gaps found in the vault

1. **No prior decision exists on image processing.** Searched for `sharp`, `srcset`,
   `object storage`, product image, upload, Lighthouse/performance — the vault has
   **zero** business rules on image dimensions, output format, derivative sizes, or
   compression targets. This spec is establishing them, not restating them. Anything
   below marked *proposed* needs human sign-off.
2. **Local disk is a deliberate current-phase choice, not an accident.** Three
   independent places say so: `FileUrlService` ("Local disk for now; replace with
   CDN/object-storage URLs later without touching callers"), `main.ts:60` ("swap for
   object storage behind FileUploadService later"), and [[Vendor Profile Customization]]
   Out-of-scope ("CDN / object-storage for images (v1 reuses local disk `uploads/`, same
   as product images)"). **No hard dependency forces an S3/CDN migration into this work.**
3. **`ProductImageDto` has nowhere to record what is stored.** The contract describes a
   single opaque URL. Any derivative-generating pipeline is invisible to the client
   without a contract change — this is the foundational blocker, not a nice-to-have.
4. **The two upload paths have drifted apart in contract shape, not in code.**
   `products/upload/multer.config.ts` and `vendors/upload/vendor-image.multer.config.ts`
   are near-identical (diffed: only the `destination` folder and the JSDoc differ —
   same `MIME_EXTENSIONS` map, same 5 MB cap, same `fileFilter`). But their *contracts*
   are not comparable: product images are a **first-class related entity** with rows and
   a DTO array, while vendor branding is **two flat nullable strings** (`logoUrl`,
   `bannerUrl` on `VendorDto`, `libs/shared/src/contracts/vendor.ts:17-18`) with no row,
   no dimensions field, and no place to hang variants. See open question 1.
5. **The extension-injection bug flagged in [[Vendor Profile Customization]] as an
   outstanding follow-up on the products path is already fixed.** `multer.config.ts`
   now derives the extension from the fixed `MIME_EXTENSIONS` map, not `originalname`.
   The vault's "Follow-ups" text is stale — no card needed for it.

## Scope
**In:**
- Pixel-dimension validation on the product-image upload path, alongside the existing
  size/MIME checks.
- Server-side resize + re-encode at upload time, before the file is persisted. The raw
  upload is **never** the file that gets served.
- A small fixed set of derivatives per uploaded image (thumbnail / card / full), plus a
  capped maximum stored dimension.
- `ProductImage` entity + `ProductImageDto` extended to record what is actually stored:
  intrinsic dimensions, byte size, and the derivative URLs.
- Web-side responsive rendering of product images (`srcset`/`sizes`, modern format with
  fallback) on the product card and the PDP hero.
- **Vendor logo/banner uploads through the same pipeline** (decision made 2026-08-16 —
  OQ1 resolved, see Open questions). Covered by PIO-5.

**Confirmed already covered, not a separate slice:**
- **Platform-fulfilled listings.** `POST /products` is guarded to both `VENDOR` and
  `ADMIN` (`products.controller.ts:33`), and `apps/web`'s admin product form
  (`admin-catalog.ts:196`) calls the identical `productsService.create()` used by the
  vendor product form. Platform-uploaded product images go through the exact same
  multer config, controller, and (once shipped) the exact same PIO-1/2/3 pipeline —
  there is no separate admin upload path to cover.
- **Every other upload surface in the codebase.** Swept `apps/api/src` for
  `FilesInterceptor`/`FileInterceptor`/`multer` (6 files, all under `products/upload` or
  `vendors/upload`) and `apps/web/src` for `input[type="file"]` (vendor-profile,
  admin-catalog, vendor-products — vendor-onboarding's own spec asserts it renders
  none). There are exactly two upload endpoints in the whole system: product images and
  vendor logo/banner. No avatar, review-image, or category-image upload exists.

**Out (explicitly):**
- **S3 / CDN / object-storage migration.** Storage stays on local disk `uploads/products`
  (and `uploads/vendors`) served via `useStaticAssets`. `FileUrlService` remains the
  single seam for a future swap. Vault research found no decision or dependency forcing
  this now (gap 2 above).
- On-the-fly / URL-parameterised resizing (`/uploads/x.jpg?w=400`). Derivatives are
  generated once at upload.
- Image editing UX (crop, rotate, reorder, re-upload). `PATCH /products/:id` still does
  not touch images.
- Orphan-file cleanup on product delete. Already absent for single files; making it
  correct for N derivatives is a separate, larger card.
- Moderation / approval of vendor imagery ([[Vendor Profile Customization]] open
  question — unchanged, still no gate).
- Backfilling existing stored images — see open question 4.

## Business rules it must honour

- **Owner-scoped writes.** `POST /products` is `@Roles(VENDOR, ADMIN)`; `vendorId` is
  resolved from the auth token, never client-supplied ([[Listing Types & Vendor Rules]],
  [[Auth & Roles]]). Processing must not introduce any path that writes another vendor's
  images.
- **Listing-type invariant untouched.** `createWithImages` enforces vendor listings have
  a vendor and platform listings do not. Image work must not reorder or bypass that.
- **`uploads/` is served statically from the API origin**, which is the same origin as
  the httpOnly refresh cookie. The stored extension must continue to come from a fixed
  mimetype allow-list, never from `originalname` — a stored-file-served-as-HTML bug here
  is an XSS against the refresh cookie ([[Auth & Roles]], and the VPC-2 incident in
  [[Vendor Profile Customization]]).
- **8 images per product** stays the cap (`FilesInterceptor(…, 8, …)` plus the explicit
  `> 8` guard in `createWithImages`). Derivatives do not count against it.
- **Primary-image semantics stay.** `isPrimary` = first uploaded; `displayOrder` = index.
  Consumers that pick `find(isPrimary) ?? images[0]` (product card, search document
  mapper) must keep working unchanged.
- **Graceful degradation is mandatory, not optional.** Images already stored have no
  derivatives and no dimensions. Every consumer must fall back to `url` when variants
  are absent, and never render a broken image or a layout jump.
- No money, inventory, or order state is touched by this feature. Standard unit-test
  discipline still applies to the new service/processor logic.

## `@hb/shared` contract impact

`libs/shared/src/contracts/product.ts` — `ProductImageDto` gains fields. **All new
fields are optional**, so existing rows/clients stay valid (this is what makes the
foundation slice shippable on its own):

```ts
export interface ProductImageVariantDto {
  url: string;
  width: number;
  height: number;
  sizeBytes: number;
}

export interface ProductImageDto {
  id: string;
  url: string;              // unchanged — the canonical/"full" URL, always present
  isPrimary: boolean;
  displayOrder: number;
  altText?: string;
  /** Intrinsic dimensions of `url`. Absent for images stored before the pipeline landed. */
  width?: number;
  height?: number;
  /** Byte size of `url`. */
  sizeBytes?: number;
  /** Responsive derivatives. Absent ⇒ render `url` alone. */
  variants?: {
    thumbnail?: ProductImageVariantDto;
    card?: ProductImageVariantDto;
    full?: ProductImageVariantDto;
  };
}
```

Not changed: `ProductDto`, `ProductCreateRequest`, `ProductUpdateRequest`, `ProductQuery`.
`VendorDto.logoUrl` / `bannerUrl` are **not** changed (out of scope pending OQ1).

Mapper to update: `ProductToResponseDto` in `apps/api/src/common/utils/mappers.utils.ts`
(the only place `ProductImage` → `ProductImageDto` happens).


### Vendor branding (PIO-4/PIO-5 — shape proposed, PIO-4 confirms it)

`libs/shared/src/contracts/vendor.ts` — `VendorDto.logoUrl`/`bannerUrl` stay flat strings
(unchanged, always present when set). Proposed addition, same optional/additive pattern
as `ProductImageDto`:

```ts
export interface VendorImageVariantDto {
  url: string;
  width: number;
  height: number;
  sizeBytes: number;
}

export interface VendorDto {
  // ...existing fields unchanged...
  logoUrl?: string;
  logoWidth?: number;
  logoHeight?: number;
  logoSizeBytes?: number;
  logoVariants?: { thumbnail?: VendorImageVariantDto; full?: VendorImageVariantDto };
  bannerUrl?: string;
  bannerWidth?: number;
  bannerHeight?: number;
  bannerSizeBytes?: number;
  bannerVariants?: { card?: VendorImageVariantDto; full?: VendorImageVariantDto };
}
```

PIO-4 is where this shape gets finalized (or revised) — PIO-5 implements whatever PIO-4
records.

## Schema impact

`product_images` table — new **nullable** columns, so no backfill is required and
`migration:run` is safe against existing data (per `apps/api/CLAUDE.md`):

- `width` `int NULL`
- `height` `int NULL`
- `sizeBytes` `int NULL`
- `variants` `jsonb NULL` — `{ thumbnail?, card?, full? }`, each `{ url, width, height, sizeBytes }`

Two migrations (one per slice), symmetric `up`/`down`. `synchronize` stays off.
Rationale for `jsonb` over a `product_image_variants` child table: the variant set is a
fixed, small, always-read-together blob owned entirely by the pipeline — same reasoning
that settled `profileSections` as `jsonb` in [[Vendor Profile Customization]].

## Proposed processing rules (need sign-off — see OQ2)

Derived from the `a2e7242` brand-asset precedent and general practice, **not** from any
existing vault decision:

| Knob | Proposed | Note |
|---|---|---|
| Reject above | 8000 × 8000 px | Absurd/decompression-bomb guard; 422, clear message |
| Auto-downscale above | 2000 px longest edge | Anything between the two is silently downscaled, not rejected |
| `full` derivative | ≤ 2000 px longest edge, target ≤ 500 KB | The canonical `url` |
| `card` derivative | 800 px longest edge, target ≤ 200 KB | Grid / carousel |
| `thumbnail` derivative | 300 px longest edge, target < 100 KB | PDP filmstrip, search results, cart lines |
| Output format | WebP primary | Precedent already accepts WebP + fallback |
| Fallback format | JPEG | Only if OQ2 says the browser matrix needs it |
| Aspect ratio | Preserved, never cropped | Cropping is a product decision nobody has made |
| Upscaling | Never | A 400 px upload stays 400 px; smaller derivatives only |
| Strip EXIF | Yes | Vendor phone photos carry GPS — privacy, not just bytes |
| Auto-orient | Yes | Honour EXIF rotation before stripping it |

## Technical notes for implementers

- **`diskStorage` vs `memoryStorage` is the central design choice of slice 2.** Today
  Multer writes to disk before any validation runs, which is why the controller pipe
  needs `fallbackToMimetype: true` (`file.buffer` is empty under `diskStorage` — the
  VPC-2 lesson in [[Vendor Profile Customization]]). Processing from a buffer would
  enable *real magic-number* validation instead of trusting the client `mimetype`, and
  avoids writing the raw original to disk at all — but costs up to 8 × 5 MB = 40 MB of
  peak heap per request. Whichever way slice 2 goes, it must not silently regress the
  `fallbackToMimetype` fix.
- **`key` stops being a single filename.** It is currently `file.filename` and is
  documented as "used for deletion". With N derivatives, deletion needs all of them.
  Nothing deletes files today, so this is not a regression — but slice 2 should keep
  `key` meaningful (e.g. the uuid stem) rather than leaving it pointing at one variant.
- **The search index carries the primary image URL** (`search-document.ts:64`). Once a
  thumbnail exists, search results should use it. Small follow-on, flagged not carded.

## Open questions (human decision required)
1. **~~Is the vendor logo/banner path in scope?~~ RESOLVED 2026-08-16: yes.** Product
   owner decided vendor logo/banner uploads go through the same optimization pipeline.
   The contract-shape difference noted below still has to be designed, not skipped: the
   two multer configs are near-identical (diffed: only the `destination` folder differs
   — same `MIME_EXTENSIONS` map, same 5 MB cap, same `fileFilter`), but `VendorDto` has
   `logoUrl`/`bannerUrl` as flat nullable strings with no row and no variant slot, unlike
   `ProductImageDto`. A logo (small, often square, sometimes needs transparency) and a
   banner (wide hero) also want different derivative presets from a product photo and
   from each other. → carded as **PIO-4** (design: contract shape + presets) unblocking
   **PIO-5** (implementation).
2. **Confirm the dimension caps, derivative sizes, output format and quality targets** in
   the table above. In particular: is a **JPEG fallback required**, or is WebP-only
   acceptable? WebP is supported by every browser from ~2020; a fallback roughly doubles
   storage and processing. The `a2e7242` precedent emitted a fallback, but that was for
   the marketing landing page, which has a broader reach requirement than an
   authenticated marketplace.
3. **Synchronous or asynchronous processing?** Synchronous keeps `POST /products` a
   single atomic call and needs no queue, at the cost of latency (8 images × resize
   ≈ 1–3 s worst case). Asynchronous needs a job runner the codebase does not have
   (`EventEmitter2` exists, used by the search indexer — could carry it, but then image
   URLs are briefly absent and the client must poll or degrade). *Recommendation: sync
   for v1 — the create-product flow already tolerates a multi-MB multipart upload; do
   not add a queue for this.*
4. **What happens to product images already stored?** Options: (a) leave them — they
   keep serving the original, `variants` stays null, the web falls back to `url`
   (zero risk, zero work, the un-optimised bytes persist); (b) a one-off backfill
   command that reprocesses existing files. *Recommendation: (a) for this feature; card
   the backfill separately only if the current catalogue is large enough to matter — a
   product decision about how much legacy data actually exists.* Same applies to
   existing vendor logos/banners once PIO-5 ships.
5. **Does the 5 MB per-file cap change?** Once uploads are downscaled server-side, a
   larger cap becomes tolerable and vendor UX improves (fewer rejected phone photos).
   Raising it also raises peak memory if slice 2 goes the `memoryStorage` route. Client
   side, `apps/web` hard-codes a 5 MB pre-check in the vendor profile editor — any
   change has to move together. *Recommendation: leave at 5 MB; revisit later.*

## Vertical slices → cards
PIO-5 created: [EgynexWb](https://trello.com/c/EgynexWb) (2026-08-16). Table below updated.

**Cards created** in the **To Do** list, verbatim from the definitions below:

| # | Title | Layer | Depends on | Card |
|---|---|---|---|---|
| PIO-1 | Product image dimension validation + `width`/`height`/`sizeBytes` on the contract | shared + api + migration | — | [8AQq2C3E](https://trello.com/c/8AQq2C3E) (2026-08-16) |
| PIO-2 | Server-side resize/re-encode pipeline producing derivatives on product upload | api + shared + migration | PIO-1 | [1f1W44bw](https://trello.com/c/1f1W44bw) (2026-08-16) |
| PIO-3 | Responsive product-image rendering (`srcset`) on card + PDP | web | PIO-2 | [0HQBCxik](https://trello.com/c/0HQBCxik) (2026-08-16) |
| PIO-4 | Design: vendor logo/banner presets + contract shape (OQ1 resolved — building it) | design/spike | PIO-2 | [wLFJ22ps](https://trello.com/c/wLFJ22ps) (2026-08-16) |
| PIO-5 | Vendor logo/banner images through the shared optimization pipeline | api + web + shared + migration | PIO-2, PIO-4 | *(created below)* |

Sequential: PIO-1 unblocks PIO-2 unblocks PIO-3. PIO-2 also unblocks PIO-4, which unblocks
PIO-5. Platform-fulfilled listings need no separate card — confirmed they ride the same
`POST /products` path as vendor products (see Scope).

### PIO-1 — Product image dimension validation + contract dimension fields

Foundation slice. Adds the image-metadata dependency, rejects absurd uploads, and makes
the contract able to describe what is stored. **No resizing yet** — this slice alone
makes the system honest about the problem; PIO-2 fixes it.

Acceptance criteria:
- [ ] `sharp` added to `apps/api` dependencies (metadata probe only in this slice).
- [ ] Upload path reads intrinsic pixel dimensions of every uploaded product image before persistence.
- [ ] Uploads exceeding the agreed max dimensions (proposed 8000 × 8000) are rejected with `422` and a message naming the actual and allowed dimensions; the partially-written file is cleaned up.
- [ ] Validation is expressed as a DTO/pipe validated with class-validator on the endpoint input, implementing the matching `@hb/shared` interface — no ad-hoc inline checks in the controller body.
- [ ] `ProductImageDto` in `libs/shared/src/contracts/product.ts` gains optional `width`, `height`, `sizeBytes`. No breaking change to existing consumers.
- [ ] TypeORM migration adds nullable `width`, `height`, `sizeBytes` to `product_images`, with symmetric `up`/`down`. `synchronize` stays off.
- [ ] `ProductToResponseDto` (`apps/api/src/common/utils/mappers.utils.ts`) maps the new fields through.
- [ ] Existing rows (null dimensions) still serialise and render — regression test covers a null-dimension image.
- [ ] Unit tests: valid image records dimensions; oversized image rejected with 422; non-image still rejected as before; `fallbackToMimetype` behaviour not regressed.
- [ ] Existing 5 MB + mimetype checks unchanged and still enforced.

### PIO-2 — Server-side resize/re-encode pipeline producing derivatives

The core slice. The raw upload stops being the served file.

Acceptance criteria:
- [ ] A reusable image-processor module (not product-specific in its API, so PIO-4/5 can reuse it) resizes and re-encodes uploaded images before persistence.
- [ ] Every uploaded product image produces the agreed derivative set (proposed `thumbnail` / `card` / `full`) at the agreed caps; the raw original is never the URL served to clients.
- [ ] Aspect ratio preserved; images are never upscaled beyond their intrinsic size.
- [ ] EXIF auto-orientation applied, then EXIF stripped (GPS/privacy).
- [ ] Stored extension continues to come from the fixed mimetype allow-list, never `originalname` — regression test asserts this.
- [ ] `ProductImageDto` gains optional `variants` (`ProductImageVariantDto` per size); `ProductImage` entity gains a nullable `variants` jsonb column via a TypeORM migration with symmetric `up`/`down`.
- [ ] `ProductImage.key` remains a meaningful handle for all derivatives of one upload (not one variant's filename).
- [ ] `ProductDto.images[].url` continues to point at a working image for both new (processed) and legacy (unprocessed) rows.
- [ ] Whichever storage mode is used (`diskStorage` vs `memoryStorage`), the `fallbackToMimetype: true` disk-storage validation fix is not regressed — covered by a test.
- [ ] Unit tests on the processor: derivative dimensions/format, no-upscale rule, aspect preservation, EXIF stripped, corrupt/truncated image handled as `422` rather than a 500, per-derivative failure does not leave a half-written product.
- [ ] The 8-images-per-product cap and `isPrimary`/`displayOrder` semantics are unchanged.
- [ ] Storage remains local disk under `uploads/products` via `FileUrlService` — no S3/CDN introduced.

### PIO-3 — Responsive product-image rendering

Acceptance criteria:
- [ ] `ProductCard` (`apps/web/src/app/shared/components/product-card/`) renders the responsive derivative set with `srcset` + `sizes` instead of a single `[src]`.
- [ ] PDP hero (`apps/web/src/app/features/product-detail/product-detail.html`) renders the responsive derivative set; the filmstrip/alternate thumbs use the `thumbnail` derivative.
- [ ] Fallback: images with no `variants` (legacy rows) render `url` in a plain `<img>` with no console error and no layout shift.
- [ ] `width`/`height` attributes set from the contract where available, to reserve layout space and avoid CLS.
- [ ] `loading="lazy"` retained on grid cards; the PDP hero is **not** lazy (it is the LCP element).
- [ ] SSR-safe — no `window`/`document` access; both routes still server-render/prerender cleanly.
- [ ] Existing alt-text behaviour (`altText ?? product.name`) unchanged.
- [ ] Vitest specs cover: variants present → correct `srcset`; variants absent → single-src fallback; primary-image selection unchanged.

### PIO-4 — Design: vendor logo/banner presets + contract shape

**OQ1 resolved 2026-08-16: vendor logo/banner uploads WILL go through the pipeline.**
This card is no longer a go/no-go spike — it's the design step that nails down the
details before PIO-5 implements them. Depends on PIO-2 (needs the reusable processor to
exist). Unblocks PIO-5.

Acceptance criteria:
- [ ] `apps/api/src/vendors/upload/vendor-image.multer.config.ts` and `vendors.controller.ts` inspected in detail and the delta from the products path written up (confirmed near-identical already — only `destination` differs).
- [ ] Logo- and banner-specific derivative presets decided and recorded (proposed starting point: logo — capped ~512px longest edge, no crop, preserve transparency; banner — wide-hero derivatives similar to the `a2e7242` LSM breakpoints, e.g. 640/960/1280/1536).
- [ ] `@hb/shared` contract shape for `VendorDto` finalized (see the proposed `VendorImageVariantDto` shape in the contract-impact section above — confirm or revise field names/nesting).
- [ ] Decision recorded on WebP-only vs WebP+fallback for vendor branding assets, consistent with whatever OQ2 settles for product images.
- [ ] PIO-5's acceptance criteria reviewed against this card's findings and adjusted if the design changed anything.
- [ ] No production code changes land on this card.

### PIO-5 — Vendor logo/banner images through the shared optimization pipeline

Implementation slice for the decision made 2026-08-16 (OQ1). Wires
`apps/api/src/vendors/upload/vendor-image.multer.config.ts` through the PIO-2 processor
using the presets PIO-4 settles on.

Acceptance criteria:
- [ ] Vendor logo and banner uploads run through the same reusable image-processor module built in PIO-2, using the logo/banner-specific presets recorded in PIO-4.
- [ ] The raw upload is never the file served for either `logoUrl` or `bannerUrl`.
- [ ] Aspect ratio preserved, never cropped; never upscaled beyond intrinsic size; EXIF auto-oriented then stripped — same rules as PIO-2.
- [ ] Stored extension continues to come from the fixed mimetype allow-list, never `originalname` — regression test asserts this (mirrors the VPC-2 lesson already fixed on this path).
- [ ] `VendorDto` gains the fields finalized in PIO-4 (proposed: `logoWidth`/`logoHeight`/`logoSizeBytes`/`logoVariants`, `bannerWidth`/`bannerHeight`/`bannerSizeBytes`/`bannerVariants`), all optional — no breaking change to existing consumers.
- [ ] TypeORM migration adds the corresponding nullable columns to the vendor table, symmetric `up`/`down`. `synchronize` stays off.
- [ ] Vendor DTO validated with class-validator implementing the updated `@hb/shared` interface — no ad-hoc inline checks in the controller body.
- [ ] Vendor-profile UI (`apps/web/src/app/features/vendor/pages/vendor-profile/`) renders the new logo/banner variants responsively where displayed (storefront + vendor profile page), falling back cleanly to the flat `logoUrl`/`bannerUrl` for vendors who haven't re-uploaded since this shipped.
- [ ] Existing vendors with only `logoUrl`/`bannerUrl` set (no variants) keep working — regression test covers the no-variants case.
- [ ] Unit tests: valid logo/banner records dimensions + variants; oversized/non-image rejected as before; `fallbackToMimetype` behaviour not regressed.
- [ ] The existing 5 MB + mimetype checks on this path are unchanged unless OQ5 (5 MB cap) is explicitly revisited here too.

## Handoff gap
Closed 2026-08-16 — all four cards (PIO-1 through PIO-4) created in **To Do**, verbatim from
the definitions above, via the Trello REST fallback (see the card links in the slice table
above). Verified against the board first: no existing card in To Do/In Progress/In Review
covers this work, so nothing here duplicates in-flight effort. Ready for `/ship-card
PIO-1`.