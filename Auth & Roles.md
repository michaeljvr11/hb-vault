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
- Frontend: `localStorage` access is platform-guarded (`isPlatformBrowser`) because of SSR. Auth state lives in `AuthService` — extend it, don't fork it.
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
- **Medium — security headers (added).** `helmet` with HSTS + conservative CSP;
  `crossOriginResourcePolicy: cross-origin` so `/uploads` images still embed from the web app.
- **Medium — Google OAuth (hardened).** Cookie-backed `state` store (login-CSRF defense
  without express-session — fits our cookie architecture) + reject sign-in unless Google
  reports the email verified (blocks account-takeover-by-linking).
- **DB structure decision (unchanged, now documented).** Single `users` table + `role` enum
  is **kept** — the real risk was the writable `role` column (the Critical above), not
  co-location. Splitting admins out adds complexity without removing the mass-assignment
  class of bug. See [[HB Domain Model]].
- **Deferred:** access-token-in-`localStorage` → in-memory (frontend refactor; CSP added as
  interim); transitive `multer@2.1.1` DoS bump (needs an isolated lockfile-regen PR).

Enforcement going forward: `public-routes.guardrail.spec.ts` pins the exact set of
`@Public()` routes (a new accidental public route fails CI). API 139/139, web 295/295, build clean.