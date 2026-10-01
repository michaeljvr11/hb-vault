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



**Correction (2026-09-30):** The dev box has a 4 GB `/swapfile` (set up 2026-09-11 during prod-migration work) and `vm.swappiness=10` in `/etc/sysctl.d/99-swappiness.conf` — the statement "no swap" above is incorrect. Production VM (separate) still requires explicit setup; commands are in the runbook.

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


---

## Implementation Notes — 2026-09-30 (OPT-4, OPT-7, OPT-10 shipped; SEC-9 shipped separately)

**Status:** Both PRs pending (OPT batch in review, SEC-9 open)

**OPT-7 (wJwlM7CN) — Node 20 → Node 22 LTS:** Upgraded from Node 20 (EOL 2026-04-30) to 22 inside Angular 21's supported range (^22.12; Node 24 not needed). node:22-slim in both Dockerfiles; CI updated (ci.yml both jobs, evidence.yml); root engines ">=22.12.0". Verified: both images build locally on Node 22.23.3, sharp 0.35.3 loads prebuilt binary and encodes WebP inside api image. Compose healthchecks use node fetch unaffected. Not yet verified: CI run, dev-box healthchecks, upload smoke on live box.

**OPT-4 (1WQrf8DY) — Memory budget (limits + tuning):** Container limits (db 1024m, meilisearch 512m, api 768m, web 384m, caddy 128m; sum 2816 m of 3.8 GB); NODE_OPTIONS --max-old-space-size=384 (api runtime) / 256 (web runtime); MEILI_MAX_INDEXING_MEMORY=256Mb. Postgres: shared_buffers=256MB, effective_cache_size=768MB, work_mem=4MB, maintenance_work_mem=64MB, max_connections=30. Runbook "Memory budget" section added. Baseline idle docker stats (2026-09-30 pre-limits): db 49 MiB, meili 81, api 75, web 56, caddy 14. FINDING: dev box already has 4 GB /swapfile (not assumed in card); runbook corrected, prod VM needs explicit setup (commands listed).

**OPT-10 (heEPpPuW) — Monitoring & alerts (disk, memory, OOM, container restart):** New `infra/healthcheck.sh` (systemd timer every 15 min as user deploy), `infra/oomwatch.sh` (always-on service writing `/opt/hb/.healthcheck/oom.log`), CI ships both units + scripts. Checks: Docker daemon + expected service containers running; disk root/docker-data 80%/90% warn/crit; MemAvailable <300/<150 MB; swap in use >=256 MB; container restart-count increase, not-running, OOM kills. Trend logs (disk, mem, uploads/pgdata bytes, image size) capped at 2000 lines; alerts via Resend (OPS_ALERT_EMAIL from `/opt/hb/.env`, read via stdin, never printed). Per-condition 4 h cooldown, escalation immediate. Tested live on dev box (local Resend stub, no real mail): send, rate limit, escalation, CRLF in env, restart detect, missing service, one-off containers ignored, OOM via watcher, failed-send retry, corrupt cooldown stamp, unreadable lock. Shellcheck clean. NOT YET DONE: no real alert mail sent (dev box has no RESEND_API_KEY / OPS_ALERT_EMAIL; owner must add + sign up for external monitor); timer/watcher units not installed (install after merge+deploy); external uptime monitor not created (human step). Key lessons recorded: docker kill leaves container exited (not auto-restarted); State.OOMKilled flaps on restart (can't detect OOMs on 15-min poll); oomwatch service needed for reliable OOM tracking; docker events replays only last ~2.5 min; MemAvailable+SwapFree can't fire with container swap disabled; bash bugs found only on live box; memory-mapped deploys need main() wrapper against corruption.

**SEC-9 (K8acFXq2) — Multer dependency hardening (separate PR, audit finding M3):** npm audit listed FIVE multer advisories; bumping @nestjs/platform-express to ^11.2.7 pins multer@2.4.0 (was <=2.2.0), fixing transitive copy. 44-line lockfile diff. Verified: npm ls multer 2.4.0 everywhere, no multer/platform-express in audit output, API suite 89 suites/1405 tests pass, real multipart smoke test via FilesInterceptor (8+ files / >5 MB rejected, aborted upload, nested field test). NOT DONE: promoting CI's npm audit to blocking deferred (9 other High advisories remain — sharp, brace-expansion, etc.); SECURITY.md/ci.yml comments updated. Note: `overrides` not used (platform-express bump was cleaner). Card deviation documented.


## Implementation Notes — 2026-10-01 (OPT-5 shipped)

**Status:** In review · Branch: `feat/MM6LVEVi-deploy-pipeline-and-api-memory` · PR: to be opened

**OPT-5 (sqEp2xXj) — API image-processing memory hardening (jemalloc, sharp limits, concurrency gate, batched reindex):**

**Dockerfile runtime stage (Node 22-slim, OPT-7 in place):** `libjemalloc2` installed; `LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2` set with `ENV` in the runtime stage, so every process in the container (including the node process) is launched with jemalloc preloaded. Build-time `test -f` guards against arm64 misarch (aarch64-linux-gnu path required on non-x86). VERIFIED locally: image builds; in-container `/proc/self/maps` of a running node process contains jemalloc, LD_PRELOAD confirmed set.

**Sharp tuning at bootstrap (`main.ts` before `NestFactory.create`):** New `apps/api/src/common/image-processing/sharp-tuning.ts` (`resolveSharpTuning()` is the pure env-to-settings step, `applySharpTuning()` applies it; called from `main.ts`). Reads env vars (default fallbacks): `SHARP_CACHE_MEMORY_MB` (32 MB), `SHARP_CACHE_ITEMS` (50), `SHARP_CONCURRENCY` (1). Calls `sharp.cache({ memory, files: 0, items })` + `sharp.concurrency(n)`. New config helpers `parseNonNegativeInt()` / `parsePositiveInt()` in `common/config/config.utils.ts` (invalid/negative input → default, no throw).

**ImageProcessorService concurrency gate:** New `apps/api/src/common/image-processing/semaphore.ts` exports a FIFO `Semaphore` class (generic, reusable). `ImageProcessorService.process()` acquires one slot before processing; excess calls enqueue (never reject). Slot released in `finally` (covers success, error, 422 validation failures). Configured via `IMAGE_PROCESS_MAX_CONCURRENT` env var (default 2). Output bytes/quality unchanged. `ConfigService` is injected `@Optional()`, so `new ImageProcessorService()` still works and the existing processor specs pass unmodified. Specs prove at-most-N-concurrent via deferred promises.

**SearchIndexerService.runFullReindex pagination:** Iterates by `id` cursor using `MoreThan(lastId)`, `ORDER BY id`, 500-row pages. Only the `Set<uuid>` of returned IDs is kept across pages for the stale-UUID diff (pruning phase). Boot-time reindex and 3am cron both use it. Specs cover 3-page scan, exact-multiple boundary, empty table, pruning correctness across pages.

**Tests:** Full API suite 92 suites / 1428 tests pass. Lint clean. Full build passes. New specs: `semaphore.spec.ts` (acquire/release, FIFO, throwing task frees its slot), `image-processor.concurrency.spec.ts` (at most N concurrent `process` calls; a 422 releases its slot), `sharp-tuning.spec.ts` (env parsing, defaults), and four paging cases added to `search-indexer.service.spec.ts` (3 pages, exact-multiple boundary, empty table, prune across pages). `config.utils.spec.ts` gained cases for the two new parsers.

**Review observations NOT fixed (follow-ups):**
- **(a) Prune race window widened.** A product created mid-reindex whose uuid sorts below the current cursor can be indexed by its event, then pruned by the stale-ID diff until its next update or 3am run. The same race existed over a single query before; a cheap guard would skip pruning docs created after reindex start time.
- **(b) Semaphore bounds libvips work but not queued multer buffers.** The semaphore gates concurrent `sharp.process()` calls but not the `memoryStorage` upload buffers (up to 5 MB each) waiting in the queue. Queue depth is unbounded.

**NOT DONE (awaits deployed dev-box image):** The card's acceptance criteria require before/after `docker stats` peak/settled measurements for a scripted burst of 5 parallel 8-image product creates, and in-container `/proc/1/maps` check on the running api to verify jemalloc. Acceptance items for these stay open. Optional follow-up: document the four env vars (`SHARP_CACHE_MEMORY_MB`, `SHARP_CACHE_ITEMS`, `SHARP_CONCURRENCY`, `IMAGE_PROCESS_MAX_CONCURRENT`) in `docker-compose.prod.yml`, `.env.example`, and the runbook (defaults apply if unset).