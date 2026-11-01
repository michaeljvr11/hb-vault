---
operator: Josh
date: 2026-09-14
session: 17
tags: [security, trello-mcp, dev-server, ssh, prod-readiness, retroactive]
---

# Session 17 — Trello MCP check, SEC/PROD human gates, dev-server SSH access

> Logged retroactively on 2026-11-01 from chat history. Followed straight on by [[session-18]] (PROD-1).

## Vault pull
Pulled three of Michael's commits: the corrected discount PATCH semantics (omitting the fields now
leaves a discount alone, and `discountPercent: null` clears it), SSR internal API routing (83G6w4yZ),
and SEC-1..SEC-4, including the correction that `helmet` never covered the SSR origin. The local
`feat/UdeL4byD-product-discounts` branch was **superseded** by the version on the base branch, so no PR was opened from it.

## Trello MCP
It wasn't loaded because the session started in the parent `TechAcc` folder, and project-scoped MCP
servers only load at their own project root. Credentials were checked against the REST API (board
"H&B E-commerce", 7 lists, 180 cards). `.mcp.json` is gitignored and has never been committed.

## SEC tickets
SEC-1..4 were done. SEC-5..8 were "In Progress" but had **not shipped**: no branches or PRs, the
Angular bump was absent, and `MEILI_MASTER_KEY` was still in the live API container. Michael was
working on them, so Josh didn't start SEC-6a.

## Dev-server SSH access
- The `SSH_HB` env var turned out to hold a **public** key (Michael's side generated a pair and sent
  the public half), so it could never authenticate.
- A long diagnosis showed the server was always fine. The client's `BatchMode=yes` hid the passphrase
  prompt on Josh's old (work-issued) key, which made it look like a rejection.
- **Fix:** a new dedicated, passphrase-less `josh-hb-dev` ED25519 key, scoped with `IdentitiesOnly`.
  Josh read the fingerprint out from his own terminal, Michael appended it, and the login worked.
  No key material is recorded here.
- **Read-only recon:** all five `hb-prod-*` containers were healthy. Only 22/80/443 are reachable
  externally, checked from both inside and outside. fail2ban and unattended-upgrades are active, and
  the SEC-2 site-auth gate is live. `/opt/hb/.env` is `root:deploy`, so SEC-8's env edit needs sudo
  (Josh's sudo needs a password). The `.env` contents were never read.

## Human gates for production
- **PROD-1:** confirm the approach, and a second hostname for the "one image, two hosts" check.
- **PROD-2:** choose a secrets manager, the off-box backup destination, the retention policy, a real
  restore drill, and migrating live secrets.
- **PROD-3:** a new Xneelo VM plus its security group, DNS, the GitHub `prod` Environment with
  required reviewers, and per-environment secrets. Michael owns the repo.
