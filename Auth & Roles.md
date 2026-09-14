# Auth & Roles

## The flow is SETTLED — do not redesign

JWT access token + **rotating hashed refresh token in an httpOnly cookie**. Global `JwtAuthGuard` + `RolesGuard`; routes opt out with `@Public`, restrict with `@Roles`. `register` issues the refresh cookie exactly like `login`. Passwords via `bcryptjs` (no native bcrypt).

## Roles (`libs/shared/src/enums/user-role.ts`)

| Role | Can |
|---|---|
| `customer` | browse, cart, checkout, view own orders/addresses |
| `vendor` | everything a customer can + manage **own** vendor profile, listings, and orders for their lines — see [[Listing Types & Vendor Rules]] |
| `admin` | manage categories, platform listings, vendor approval/suspension, all orders |

## Rules for agents

- New protected endpoints: rely on the global guards + `@Roles`; never hand-roll auth checks in controllers.
- **Role is never client-settable.** Self-registration is always `customer`; elevation happens only via the admin-only `PATCH /admin/users/:id/role` or vendor onboarding. New self-service DTOs must not accept `role`/`isActive`/`vendorId`. See repo-root `SECURITY.md` + the 2026-07-02 audit note below.
- Ownership checks live in the service layer (e.g. vendor editing own product) — keep that pattern.
- Frontend: `localStorage` access is platform-guarded (`isPlatformBrowser`) because of SSR. Auth state lives in `AuthService` — extend it, don't fork it. **Access token is NO LONGER stored in localStorage** (moved to in-memory only per SEC-7); other app state like consent/analytics still legitimately use localStorage.
- Secrets stay in `apps/api/.env` (gitignored). Frontend env files hold only `apiBaseUrl` + flags.

## To implement:

- Email verification / password reset flows: Password reset and email verification to be implemented using Resend. email verification not required to access site, but required to make orders. new field - isVerified to be added to user if not in place. Set everything up to send email as no-reply@hb-ecommerce.com and to the user. Resend Api token will be provided.
- Session lifetime / refresh-token expiry policy for production: Implement a small remember me checkbox on the login/register pages - this determines refresh  token longetive if ticked - then 30 days, if not 24 hours. Access Token  — short lived (15min–1h)  — stored in memory. Refresh Token — long lived (24h or 30–90 days) — stored in httpOnly cookie
- Social Auth/Login - implement with google social auth as an option for login and registration.


Related: [[HB Domain Model]]


## Implementation notes

### Login & Registration pages — 2026-06-13 (card mMFxZIKE, [PR #1](https://github.com/michaeljvr11/hb-mono-repo/pull/1))

Frontend-only rework: the login/register screens were rebuilt to the Stitch **"Login"**/**"Register"** designs (saved under `docs/design/login/` + `docs/design/register/`) as SSR-safe Angular standalone components. The settled backend auth and the `AuthService` wiring (`withCredentials`, `access_token` + `user`) were already correct and left unchanged.

- **Register uses a single "Full Name" field** (per the design) and splits it into the optional `firstName`/`lastName` of `RegisterRequest` on submit (first token → `firstName`, remainder → `lastName`). The terms-acceptance checkbox and "remember me" were dropped (absent from the Stitch designs).
- **Not-yet-built flows are visible but inert:** "Forgot Password", Google sign-in/up and the Terms/Privacy links render as "coming soon" snackbars — they make **no** API calls. These stay TBD (see the TBD section above) until a dedicated card builds them.
- Shared auth CSS primitives now live in global `apps/web/src/styles.scss` (`.auth-screen` scope); Inter + Material Symbols fonts load in `index.html`. Design tokens seeded in `docs/design/DESIGN.md` from the Stitch "Trans-Frontier Commerce System" system (primary `#015300`, Inter, 8–12px radius).


### Password reset, email verification & remember-me — 2026-06-15 (card mMFxZIKE, [PR #1](https://github.com/michaeljvr11/hb-mono-repo/pull/1), review round 2)

PR-review round implementing the "To implement" items the reviewer called out (password reset + refresh-token life), plus email verification.

- **Remember me / refresh-token life:** checkbox on login & register drives refresh longevity — **30 days** when ticked, **24 hours** otherwise (`AuthService.REFRESH_TTL`). `rememberMe` is encoded in the refresh JWT so it survives rotation; the httpOnly cookie `maxAge` mirrors it. Access token stays short-lived (`JWT_EXPIRATION=15m`). Shared: `rememberMe?` added to `LoginRequest`/`RegisterRequest`.
- **Password reset (Resend):** `POST /auth/forgot-password` + `/auth/reset-password`. Raw token emailed; **SHA-256 hash** stored (`passwordResetTokenHash` + `passwordResetExpires`, 1h); single-use; reset clears the token and drops the refresh session. `forgotPassword` returns the same message whether or not the email exists (no enumeration). Pages: `/forgot-password`, `/reset-password`.
- **Email verification (Resend):** `isVerified` column (migration `1781740800000-AuthVerificationAndReset`); verification email on register (best-effort); `POST /auth/verify-email` + `/auth/resend-verification`; `emailVerificationTokenHash` + `emailVerificationExpires` (24h). Page: `/verify-email` (verifies on `afterNextRender`, browser-only). **Order-gating on `isVerified` is NOT yet enforced** — that rule belongs with the orders module (still skeleton); follow-up card.
- **Mail:** `MailModule`/`MailService` via `resend`. Reads `RESEND_API_KEY` / `MAIL_FROM` (default `no-reply@hb-ecommerce.com`) / `APP_WEB_URL` from config; **no-ops with a warning when the key is absent** so dev/CI run without email infra. Real key lives in `apps/api/.env` (gitignored) — never committed.
- **Still TBD:** Google / social auth (separate card).

Tests: api 12/12, web 29/29; full build clean. Requires `npm run migration:run` before the new flows work against a live DB.


### Google social sign-in — 2026-06-15 (card mMFxZIKE, [PR #1](https://github.com/michaeljvr11/hb-mono-repo/pull/1), review round 3)

Implements the last "To implement" auth item (social/Google). **Social auth is no longer TBD.**

- **Flow:** server-side OAuth Authorization Code via `passport-google-oauth20`. `GET /auth/google` -> consent; `GET /auth/google/callback` -> `AuthService.validateOAuthLogin` finds-or-creates a **verified** user (existing accounts linked by email since Google verifies it; new accounts get a random, unused password to satisfy NOT NULL), issues JWT + httpOnly refresh cookie, then redirects to the web `/auth/callback`.
- **No tokens in the URL:** the callback only sets the refresh cookie; the web `/auth/callback` page exchanges it for an access token via `/auth/refresh` (browser-only, `afterNextRender`).
- **Frontend:** the Google buttons on login & register are now SSR-safe anchors to `{apiBaseUrl}/auth/google`.
- **Graceful without config:** `GoogleStrategy` boots with placeholder creds when `GOOGLE_CLIENT_ID/SECRET` are unset, so the app still runs; the route just won't complete a real sign-in. Env: `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_CALLBACK_URL` in `apps/api/.env`.

Tests: api 15/15, web 32/32; build clean.


### Refresh cookie SameSite — topology-driven config — 2026-06-18 (card mMFxZIKE)

The refresh cookie's `SameSite` attribute is now env-driven via `REFRESH_COOKIE_SAMESITE` (`apps/api/.env`), resolved by the pure `resolveRefreshCookieSecurity` (`apps/api/src/auth/refresh-cookie.ts`) and applied in `AuthController.setRefreshCookie`. **Default is `strict` — current behaviour is unchanged.** This governs **every** refresh flow (login / register / refresh + the Google callback), not just Google.

- **Why now:** Google sign-in made post-redirect token exchange a first-class path — `GET /auth/google/callback` sets the cookie and 302s to the web `/auth/callback`, which exchanges it via `POST /auth/refresh` (a same-document XHR). `SameSite` keys on the **registrable domain (eTLD+1)** of the API cookie vs. the initiating page, so deploy topology decides whether the cookie rides that XHR.
- **Same-site / same-origin (recommended):** web + API share a registrable domain (e.g. `app.hb.co.za` + `api.hb.co.za`), or the API is reverse-proxied under the web host (same-origin, also simplifies CORS). Keep `strict` — strongest CSRF posture. No change required.
- **Cross-site (different registrable domains):** `strict` (and `lax`) cookies are dropped on the refresh XHR and on page-load resumption → sessions can't resume. Set `REFRESH_COOKIE_SAMESITE=none`; the helper then **forces `Secure` on** (browsers reject `None` without it; `Secure` is always on in prod regardless). **Gate: add CSRF hardening before shipping cross-site** — `/auth/refresh` is the only cookie-authed route, and `None` exposes it to forged cross-site POSTs. CORS `credentials:true` + the JSON content-type already stop an attacker *reading* the rotated token, but the rotation itself would still fire.
- **Production topology is still undecided**, so the config is switchable per environment — the call can be made at deploy time with no code change.

Tests: `refresh-cookie.spec.ts` (6 cases: default / strict-prod / lax / none-forces-secure / whitespace / invalid-falls-back-to-strict); full auth suite 20/20. `.env.example` documents the knob.


### Security audit & hardening — 2026-07-02 (branch `security/audit-2026-07-02`)

Full security pass over auth / authZ / DB / web-security. Findings report lives at
`docs/security/AUDIT-2026-07-02.md`; the auth model + new-route checklist now live in
repo-root `SECURITY.md`. Most of the auth core reviewed as **already solid** (bcrypt-12,
hashed single-use reset tokens, no user enumeration, rotating hashed refresh cookie,
global-guard authZ, service-layer ownership, parameterized queries). Changes made:

- **Critical — role mass-assignment on register (fixed).** `POST /auth/register` accepted
  a validated `role` field, so anyone could self-register as `admin`. Removed `role` from
  `RegisterRequest` / `RegisterDto` / the web form and force `CUSTOMER` in `AuthService`.
  **New settled rule:** role is *never* client-settable — elevation only via the admin-only
  `PATCH /admin/users/:id/role` or vendor onboarding.
- **High — admin-bootstrap race (fixed).** `POST /auth/bootstrap-admin` is now gated by
  `ADMIN_BOOTSTRAP_SECRET` (constant-time compare); **required in production** (endpoint
  disabled if unset), zero-config in dev. This supersedes the "self-sealing is enough"
  assumption — self-sealing left a first-caller-wins window on a fresh prod DB.
- **High — rate limiting (added).** `@nestjs/throttler` global default (120/min/IP) + tighter
  per-route limits on login/reset/verify/register/forgot/bootstrap. `trust proxy = 1` so
  per-IP limits use the real client IP without X-Forwarded-For spoofing.
- **Medium — security headers (added — `/api/*` and `/uploads/*` only).** `helmet` with HSTS +
  conservative CSP; `crossOriginResourcePolicy: cross-origin` so `/uploads` images still embed
  from the web app.
  **Scope correction (2026-09-11):** this line overstated the delivery for three months.
  `helmet` runs inside the NestJS process, which serves only those two prefixes. The Angular
  SSR app is a **separate container** behind the same hostname and never received any of it.
  Measured live on 2026-09-11: `/api/health` returned the full helmet suite; `/shop` returned
  only HSTS, `X-Content-Type-Options` and `Referrer-Policy` (set by Caddy) — no CSP, no
  `X-Frame-Options`, no `Permissions-Policy`, plus a leaked `X-Powered-By: Express`. See the
  SEC-1..SEC-4 note below.
- **Medium — Google OAuth (hardened).** Cookie-backed `state` store (login-CSRF defense
  without express-session — fits our cookie architecture) + reject sign-in unless Google
  reports the email verified (blocks account-takeover-by-linking).
- **DB structure decision (unchanged, now documented).** Single `users` table + `role` enum
  is **kept** — the real risk was the writable `role` column (the Critical above), not
  co-location. Splitting admins out adds complexity without removing the mass-assignment
  class of bug. See [[HB Domain Model]].
- **Deferred:** access-token-in-`localStorage` → in-memory (frontend refactor; recorded here
  as "CSP added as interim" — **that interim control did not exist**. The CSP was helmet's,
  scoped to `/api/*`; the HTML origin, where XSS actually runs, had none. The item was carried
  as mitigated while carrying no mitigation. SEC-1 adds the static headers including
  `frame-ancestors 'none'`; SEC-5 will add the real CSP and genuinely restore the interim
  control. The refactor itself is still open either way); transitive `multer@2.1.1` DoS bump
  (needs an isolated lockfile-regen PR).

Enforcement going forward: `public-routes.guardrail.spec.ts` pins the exact set of
`@Public()` routes (a new accidental public route fails CI). API 139/139, web 295/295, build clean.


### Security guards batch — SEC-1..SEC-4 — 2026-09-11 (cards BGykPFS7, Jw33F6CV, vKRja5fR, NSSA9AXv; branch `feat/BGykPFS7-security-guards-batch`)

Four cards from a 2026-09-11 pre-production review, shipped as one PR. Production does not
exist yet; there is one dev box at `https://dev.hb-ecommerce.com`. Production will be a
separate VM.

**The headline correction.** The 2026-07-02 audit recorded "security headers (added)" and
"CSP added as interim" against the `localStorage` access token. Both overstated the delivery.
`helmet()` runs inside the NestJS process, which serves only `/api/*` and `/uploads/*`; the
Angular SSR app is a separate container behind the same hostname and got none of it. Measured
live on 2026-09-11: `/api/health` carried the full helmet suite, `/shop` carried only HSTS,
`X-Content-Type-Options` and `Referrer-Policy` from Caddy — no CSP, no `X-Frame-Options`, no
`Permissions-Policy`, and a leaked `X-Powered-By: Express`. **Clickjacking was unmitigated on
login, checkout, vendor and admin for that whole period**, and the XSS mitigation recorded
against the `localStorage` token did not exist where XSS runs. The two audit lines above are
corrected in place.

- **SEC-1 — static security headers on the HTML origin (`Caddyfile`).** `Content-Security-Policy
  "frame-ancestors 'none'"`, `X-Frame-Options DENY`, `Permissions-Policy` (camera, microphone,
  geolocation, payment all empty), `-X-Powered-By`. Applied at the Caddy edge because that is
  the only layer in front of both containers. **Scoped to the web `handle`, not site-wide**:
  the site-wide block uses `defer`, which *replaces* rather than appends, so a site-wide CSP
  would overwrite helmet's full policy on `/api/*` with nothing but `frame-ancestors` — a
  straight downgrade. `payment=()` must be revisited when a payment provider is chosen.
- **SEC-2 — non-production access gate (`Caddyfile`, `docker-compose.prod.yml`, `infra/deploy.sh`).**
  HTTP basic auth at the edge plus `X-Robots-Tag: noindex, nofollow`, both env-driven
  (`SITE_AUTH_ENABLED`, `SITE_AUTH_USER`, `SITE_AUTH_HASH`, `SITE_NOINDEX`); production leaves
  them unset and inherits nothing. Basic auth was chosen over `noindex` alone because `noindex`
  relies on crawlers honouring it and does nothing about anyone holding the link. `/api/health`
  is exempt (exact path match) so uptime checks and CI's probe keep working. The `noindex`
  header is deliberately independent of the gate, so it still applies if the gate is lifted for
  a demo.
- **SEC-3 — CI secret scanning & dependency reporting (`.github/workflows/ci.yml`,
  `.gitleaks.toml`, `.github/dependabot.yml`).** gitleaks (pinned version + sha256-verified
  binary, full history via `fetch-depth: 0`, `--redact`) **blocks**, and the image job depends
  on it so a hit cannot reach GHCR or the box. `npm audit --audit-level=high` **reports only** —
  non-blocking on purpose, because the known transitive `multer` advisory has no clean fix and
  a gating audit would wedge every PR on a finding nobody can action. Promote to blocking once
  that is resolved. Dependabot added for npm + GitHub Actions.
- **SEC-4 — CORS fails closed in production (`apps/api/src/common/config/config.utils.ts`).**
  With `NODE_ENV=production` and `CORS_ORIGINS` unset the API now refuses to boot instead of
  falling back to development defaults. This matches the existing pattern in the stack:
  `MEILI_MASTER_KEY` has no fallback, `ADMIN_BOOTSTRAP_SECRET` disables its endpoint when unset,
  the web app validates `INTERNAL_API_BASE_URL` at boot. The old fallback list contained
  `https://hnb.co.za`, inherited from the pre-merge frontend; the owner confirms it is not a
  domain we own, and it is removed. It was never exploitable on the dev box — verified live that
  the origin is rejected there, so `CORS_ORIGINS` is set — but the fallback failed *open*, onto
  a third party, with `credentials: true`. Whitespace-only counts as unset (an empty allowlist
  rejects every browser origin and is harder to diagnose than a refusal to boot). A regression
  guard asserts the defaults contain no non-localhost origin. This supersedes audit item L6,
  which recorded it as "confirm the env var is set" rather than enforcing it.

**The SEC-2 gate must fail OPEN.** CI cannot write `/opt/hb/.env` by design, so a gate that
fails closed takes the whole site down on any misconfiguration. Code review found four
reproducible ways to do exactly that, across two passes. Two were subtle:

- `docker compose` interpolates env-file values, so an **unquoted bcrypt hash is silently
  mangled**. It survives roughly one time in five depending on the salt, so it can appear to
  work and then break on the next rotation. The hash must be single-quoted in `/opt/hb/.env`.
- Caddy's `{$VAR:default}` uses **set-semantics**, so an empty value defeats the default rather
  than falling back to it.

The Caddyfile matcher is therefore defensive three ways (`== "1"` exactly, never arm the inert
placeholder hash, bcrypt shape check). But `basic_auth` parses its hash at config **load** time
regardless of whether its matcher ever fires, so a malformed hash kills the edge whether the
gate is on or off — the matcher cannot save it. `infra/deploy.sh` is the only defence: it reads
the hash back *after* compose interpolation and refuses to deploy rather than let Caddy
crash-loop. It also blocks a deploy when `CORS_ORIGINS` is missing, since the API would
otherwise fail `up -d --wait` after the old container had already been replaced.

**Split out deliberately, not shipped:** SEC-5 (`YYjbfc65`) — nonce-based CSP for the HTML
origin. It risks breaking Angular hydration and needs its own live-browser verification pass.
When it lands it must **merge** its directives into the existing `Content-Security-Policy`
header rather than emit a second one; browsers intersect multiple CSP headers, which makes the
effective policy hard to reason about.

Tests: API 1053/1053, lint clean, full build clean, gitleaks clean over full history, and 33/33
behavioural checks against real Caddy driven through real `docker compose` env resolution.

**PR:** open on branch `feat/BGykPFS7-security-guards-batch`.

**Still open (as of this 2026-09-11 batch):** SEC-5 (nonce CSP); the `localStorage` → in-memory
access-token refactor; confirming `CORS_ORIGINS` is present in `/opt/hb/.env` before merge (now
enforced by `deploy.sh`, but a missing value blocks the deploy). **Both SEC-5 and the
access-token refactor (SEC-7) landed 2026-09-14 — see the entry immediately below.**


### Access token to in-memory storage — SEC-5 & SEC-7 — 2026-09-14 (cards YYjbfc65, XodVNFmi; branch `feat/YYjbfc65-sec-cleanup-batch`)

**SEC-7 core:** JWT access token moved from `localStorage` (XSS-readable) to an in-memory field in `AuthService` (`providedIn: 'root'`), matching the vault's own settled "Auth & Roles" design. Session recovery on page load now happens via `POST /auth/refresh` against the httpOnly refresh cookie (server-verified, secure). SSR stays anonymous (no cookie jar, every auth-required route already client-render-only). **Two review rounds caught real gaps:** (a) a "Concurrency" requirement missed on first read — rotating a single-slot refresh token on every page load made concurrent browser tabs silently log each other out. Fixed with a shared single-refresh choke point plus cross-tab coordination via the Web Locks API. (b) A follow-up interceptor bug where a 401 on `/auth/login` itself was being swallowed and replaced with a refresh error, and a failed refresh wasn't clearing stale auth state — fixed and tested.

**SEC-5 parallel:** HTML origin (`apps/web/src/server.ts`) now generates a per-request nonce, sets a real CSP header with no `unsafe-inline` anywhere, threads the nonce into Angular's SSR render via a `CSP_NONCE` DI token, and patches any inline `<script>`/`<style>` tag Angular didn't nonce itself. Removed the duplicate `Content-Security-Policy` line from the `Caddyfile` (its `header`+`defer` replaces rather than appends, so it was stomping Express's policy). **Review round caught three bugs:** stale response truncation on 10 prerendered routes (stale `Content-Length` after body patching), stale-nonce CSP violations on 304 browser replays (stale `ETag`), and legitimate inline `style="..."` bindings being blocked (fixed by adding `style-src-attr 'unsafe-inline'`, which does NOT reopen the `<script>`/`<style>` XSS vector this card exists to close). Live-browser verification: zero CSP violations, hydration + event-replay confirmed.

**Tests:** API 78 suites/1061 tests, Web 89 files/1326 tests. Full build clean.

**Branch:** `feat/YYjbfc65-sec-cleanup-batch` (batched with SEC-6 Angular patch bump guardrail + SEC-8 Meilisearch scoped keys; PR pending).