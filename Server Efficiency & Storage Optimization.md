# Server Efficiency & Storage Optimization

Spec written 2026-09-25 from a full-repo audit against small-VPS best practice. Cards: **OPT-1 … OPT-10** on the Trello "To Do" list.

## Why

The stack runs on one small Xneelo VM (2 vCPU, 3.8 GB RAM, **no swap**, 38 GB disk; see `docs/runbooks/dev-server.md`). Budget is tight. The owner's main worry is disk filling up as image uploads grow. The rule for this work is to improve efficiency **without lowering image quality**.

## What is already good (do not redo)

The image pipeline ([[Product Image Optimization Pipeline]], PIO-1…5) is already best practice:
- The raw original never touches disk (multer `memoryStorage`).
- Output is WebP only, with per-preset size targets. Products: full 2000px ≤500KB, card 800px ≤200KB, thumbnail 300px ≤100KB. Vendor logo and banner use their own smaller sets.
- EXIF auto-orient, metadata stripped, no upscaling, duplicate derivatives skipped.
- Decompression-bomb guard (8000×8000) and a 5 MB upload cap.
- The web app renders `srcset` from the variants, so browsers download the smallest variant that fits.
- Other good defaults already in place: Caddy `encode zstd gzip` and HTTP/3; SSR static assets at `maxAge: 1y`; images built in CI and pulled from GHCR, never built on the box; `docker image prune` after each deploy.

## Storage maths

Worst case is about 800 KB per product image (sum of the three targets). Realistic is about 250–450 KB. At roughly 25 GB usable that is **about 55k–100k images**, which is about 20k+ products at 3–4 images each. **Images alone are not the near-term disk risk.** These are:
1. **Orphaned files.** Nothing is ever unlinked on product/vendor delete or logo/banner replace. Card 198 (image management on edit) will add a delete-image path and make this worse.
2. **Unbounded container logs.** No `logging:` limits on any service; Docker's `json-file` default grows forever.
3. **Docker image churn** (about 7 days of per-commit images; each image also carries the other app's dependencies).
4. **Unbounded `analytics_events`** (one row per tracked event, 3 indexes, no retention).
5. **Backups that will land on the same disk** once PROD-2 schedules them.

## Memory maths

3.8 GB with no swap. Postgres, Meilisearch, API (Node + sharp/libvips), SSR (Node) and Caddy run with no per-container limits and no Node heap caps. A burst of 8-image uploads (40 MB of buffers, and sharp decodes up to 8000² px) at the same moment as a Meilisearch reindex can trigger the kernel OOM killer. The OOM killer tends to pick the largest process, which is often Postgres. sharp's docs say glibc's allocator is "unsuitable for long-running, multi-threaded processes" and recommend jemalloc. `node:20-slim` is glibc.

## Decisions

- **Locked (unchanged):** WebP-only, preset sizes, 5 MB cap. AVIF was considered and rejected for now: it gives about 20–30% smaller files but encodes several times slower on 2 vCPU, and it would reopen a PIO decision.
- **Locked:** fix disk hygiene before paying for more disk or storage.
- **Owner decision (OPT-8):** object-storage provider for uploads. Cloudflare R2 (no egress fees, 10 GB free tier), Backblaze B2, or a South Africa-hosted S3-compatible option. Latency to NA/ZA users and POPIA data residency are both factors.
- **Owner decision (OPT-9):** analytics raw-event retention window (proposed 90 days raw, daily rollups kept forever).

## Cards

| Card | Scope | Priority |
|---|---|---|
| OPT-1 | Unlink image derivatives on delete/replace + reconciliation sweep | P1 |
| OPT-2 | Serve `/uploads` from Caddy with immutable caching | P2 |
| OPT-3 | Cap container logs stack-wide | P1 (quick) |
| OPT-4 | Memory budget: swapfile, container limits, Node heap caps, Meili + Postgres tuning | P1 |
| OPT-5 | API image-processing memory hardening (jemalloc, sharp limits, concurrency gate, batched reindex) | P2 |
| OPT-6 | Per-app slim Docker images + keep-last-N image retention | P3 |
| OPT-7 | Upgrade Node 20 (EOL 2026-04-30) to Node 22 LTS | P2 |
| OPT-8 | Uploads to S3-compatible object storage (depends on OPT-1) | P3, owner gate |
| OPT-9 | `analytics_events` retention + daily rollup | P3, owner gate |
| OPT-10 | Cheap disk/memory/uptime alerting | P2 |

Related: PROD-2 (backups). The `uploads` volume is not in PROD-2's scope today and has no backup. OPT-8 would fix that; until then PROD-2 should include it. Related: card 178 (Caddy access logs), which first flagged the stack-wide log problem.

## Implementation Notes — 2026-09-29 (OPT-1, OPT-2, OPT-3 shipped)

**Status:** In review · Branch: `feat/jh8htgd7-uploads-lifecycle` · PR pending

**OPT-1 (Trello gwjmMnra) — Image file cleanup on delete/replace:** New `ImageFileCleanupService` (`apps/api/src/common/image-processing/image-file-cleanup.service.ts`) resolves stored URLs to paths via `FileUrlService` and deletes them after DB changes commit. Path-safe (only plain files directly inside `uploads/<folder>`; foreign origin/other folder/`..`/nested paths ignored). Best-effort (logs warnings, never throws). Wired into product delete, vendor delete, vendor logo/banner replace, and per-image delete/replace. One-off reconciliation command `node dist/database/sweep-orphan-uploads.js` (npm run uploads:sweep:prod): dry-run by default (count + bytes), `--delete` removes, `--grace-minutes` default 60 skips young files, matches references by BASENAME (so PUBLIC_API_URL changes don't orphan files), reads product_images url+variants and vendors logo/banner url+variants and verificationDocumentUrl, lists and stats both folders before deleting. Documented in `docs/runbooks/dev-server.md` ("Orphaned upload sweep"). Not yet run on the dev box (awaits merged+deployed image). Note: vendor delete with products fails on the FK (products.vendorId has no ON DELETE CASCADE) so only logo/banner cleanup runs there. 

**OPT-2 (M2Ses1NH) — Serve `/uploads` from Caddy with immutable caching:** `/uploads/*` served by Caddy (`root * /srv` + `file_server`, read-only mount) from the `uploads` volume at /srv/uploads; SEC-2 gate still applies first (same route block). `Cache-Control: public, max-age=31536000, immutable` set only when the file exists (a `file` matcher) — unconditional headers would cache 404s. `Cross-Origin-Resource-Policy: cross-origin` set by Caddy (helmet removed from path). Safe only because filenames are `<uuid>-<preset>.webp` and never rewritten in place (upload and replace mint new uuids); **KNOWN EXCEPTION:** dev seeds overwrite fixed `seed-<slug>-<n>-*` placeholder names — rename if art/presets change. Verified against Caddy 2.11.4 with repo Caddyfile and gate armed. Box-side verifications (curl with gate cookie, API access log silent, docker inspect rw=false, browser cache repeat-visit) pending deploy.

**OPT-3 (uya5U4G8) — Logging stack-wide caps + journald size limit:** Per-service `logging:` blocks in docker-compose already merged in PR #101 (json-file, ~190 MB stack ceiling per service, not shared pool). This batch adds: runbook documents why json-file was kept over `local`; host journald capped LIVE on hb-dev-server (2026-09-29) via drop-in `/etc/systemd/journald.conf.d/hb-size-cap.conf` `SystemMaxUse=200M` (was 139.4 MB, no prior cap; default 10% of fs); runbook documents where-logs-live and total ceiling (200 M journald + ~6× 190 MB docker per-service = ~1.3 GB). Production is a separate VM: repeat the journald step there. `docker inspect ... LogConfig` verification on the box pending deploy that includes PR #101.
