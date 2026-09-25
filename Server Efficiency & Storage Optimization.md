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
