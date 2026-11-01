---
operator: Josh
date: 2026-07-17
session: 14
tags: [spec-feature, analytics, meilisearch, dev-environment, retroactive]
---

# Session 14 — Analytics spec + Meilisearch dev setup

> Logged retroactively on 2026-11-01 from chat history. This is the morning groundwork for
> [[session-10]] (the analytics `/ship-batch`).

## Analytics spec (`/spec-feature`)
Turned the request "Google Analytics on the front end plus bespoke analytics tied to checkout,
surfaced in admin dashboards and per vendor" into requirements.
- Wrote the spec note [[Analytics & Reporting]]. It covers both pillars (GA4 storefront plus bespoke
  checkout events), both surfaces (admin and vendor portal), POPIA, currency handling, and the
  `@hb/shared` contract impact.
- Created five Trello cards in dependency order: LKIGKUVM (contracts + ingestion + migration),
  FRtcp2dg (checkout instrumentation), p0kVZMii (GA4, SSR-safe, consent-gated), UC508IB6 (admin
  aggregation + widgets) and ezkaNtbZ (vendor-scoped analytics).
- Open questions raised before shipping: the GA4 measurement ID and property owner, the missing
  POPIA consent banner, the vendor revenue definition (gross GMV vs net of commission), whether
  checkout was already on `main`, overlap with ADR-001, and where the anonymous session id comes from.
  All were answered during [[session-10]].

## Meilisearch on this machine
Michael's Meilisearch search service (PRs #28, #29) was already merged and the local base branch was up to date.
- `docker-compose.yml` requires `MEILI_MASTER_KEY` with no default, so a random dev key was generated
  and written to the local env files. The values were not printed.
- `docker compose up -d` added the `meilisearch` container (v1.15) on 7700, and its health check passed.
- Migrations ran, including `SearchSynonyms` (5 seed rows). The API logged that Meilisearch was
  reachable and the synonym settings were applied. `GET /api/search?q=takkies` returned a valid, empty result.
- `SearchIndexerService` logs "Index products not found" on boot. This is expected: the index is
  created lazily on the first product write or reindex.
