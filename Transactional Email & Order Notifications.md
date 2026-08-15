# Transactional Email & Order Notifications

Status: **TE-1/TE-2/TE-3 built (2026-08-15) · TE-4 outstanding** (spec'd 2026-08-11)
Related: [[Order State Machine]] · [[Auth & Roles]] · [[Vendor & Admin Portals]] ·
[[Money & Currency Rules]] · [[Listing Types & Vendor Rules]] ·
[[Vendor Earnings & Commission]] · [[HB Domain Model]]

## Problem

Four related asks about outbound transactional email:

1. When an order is **confirmed and paid**, notify (a) the vendor(s) with lines in that
   order and (b) a platform-level operational address. Neither happens today.
2. An **admin** field to configure which address receives platform-level order/payment
   notifications.
3. A **vendor**-configurable address for their own order/payment notifications.
4. A **branded, reusable HTML template** used by most/all outbound mail — existing emails
   migrate onto it, not just the new ones.

## Current state (verified in code, 2026-08-11)

### The mail layer

`apps/api/src/mail/mail.service.ts` (71 lines) is the whole email system.

- **Provider: Resend** (`resend` ^4.8.0, `apps/api/package.json:49`). Not SMTP, not
  SendGrid. Config from `apps/api/.env.example:36-40`: `RESEND_API_KEY`,
  `MAIL_FROM` (default `no-reply@hb-ecommerce.com`), `APP_WEB_URL`.
- **No templating of any kind.** Both emails are inline HTML string literals built
  in the method body (`mail.service.ts:24-26` and `:35-37`) — two or three bare `<p>`
  tags. No layout, no logo, no header/footer, no plain-text alternative, no `reply-to`.
  There is no template engine dependency in the repo (no handlebars, no mjml, no
  nodemailer).
- **`private send(to, subject, html)`** (`mail.service.ts:40-53`) is the single transport
  seam. It takes **one** recipient string — no `cc`, `bcc`, or recipient array.
- **No-op when unconfigured**: when `RESEND_API_KEY` is absent it logs a warning and
  returns (`:42-45`), so CI and fresh checkouts run without mail infra. This behaviour
  must be preserved — the existing spec `mail.service.spec.ts` asserts it.

### Every email the system sends today — all two of them

| Email | Trigger | Call site |
|---|---|---|
| Password reset | `POST /auth/forgot-password` | `apps/api/src/auth/auth.service.ts:182` |
| Email verification | register + `POST /auth/resend-verification` | `apps/api/src/auth/auth.service.ts:224` (via `issueEmailVerification`, called from `:216` and from register) |

That is the complete list. A repo-wide grep for `MailService` finds only
`auth.service.ts`, the mail module itself, and specs.

**So the gap is wider than the brief assumed:**

- The **customer receives no order-confirmation email**. Nothing fires on payment success
  for anyone. (See Open Question 1 — deliberately NOT carded here.)
- **No vendor** email on any event — not order lines, not vendor approval/rejection
  (`VendorsService.updateStatus`), not onboarding.
- **No platform/ops** email on anything.

The vault already recorded this gap twice, consistently: [[Product Wishlist]] out-of-scope
says "no email/notification provider is wired beyond the existing verify/reset token flow",
and [[Customer Profile]] repeats it for password-change notification. Those notes are
still accurate.

### Where "confirmed and paid" becomes true

`OrdersService.capturePayment` — `apps/api/src/orders/orders.service.ts:319-357`, private,
called from `create()` at **line 264**.

- `:327` reads provider status → `:329-333` maps to `PaymentStatus`
- `:335-344` persists the `Payment` row
- `:346-351` — **the exact hook point.** On `PaymentStatus.PAID` it asserts the
  `pending → confirmed` transition and saves the order.
- `:352-356` — non-paid leaves the order `pending` and logs a warning.

This matches the state-machine trigger documented in [[Order State Machine]]
("`pending` → `confirmed` … payment authorized/paid").

`OrdersService.updateStatus` (`:277-315`) is the other transition gateway; it is not
part of this feature's scope.

### There IS already an event mechanism — reuse it, do not invent one

`@nestjs/event-emitter` ^3.1.0 is a dependency (`apps/api/package.json:29`) and
`EventEmitterModule.forRoot()` is registered (`apps/api/src/app.module.ts:36`).

The house pattern lives in `apps/api/src/common/events/domain-events.ts` — a namespaced
event-key const plus a typed payload interface. Its header comment states the design
intent verbatim: *"In-process domain events (EventEmitter2). Best-effort by design."*
Emitters today: `ProductsService`, `VendorsService`. Listener:
`SearchIndexerService` (`apps/api/src/search/search-indexer.service.ts:51,68`), whose
own doc comment sets the precedent this feature must copy:

> "Indexing failures are logged and swallowed: they must never break the originating
> Postgres write."

Notification dispatch is the same shape of problem and gets the same treatment.

### Admin settings — nothing exists

There is **no** `PlatformSettings` / config entity or table. A grep for
`settings|PlatformConfig` hits only `SearchSettingsService`, which configures the
Meilisearch index and is unrelated.

Closest precedent for admin-owned config: **`commission_rates`**
(`apps/api/src/commission/entities/commission-rate.entity.ts`, migration
`1784419200000-CommissionRates.ts`, controller `apps/api/src/commission/commission.controller.ts:9`
at `admin/commission-rates`, admin page
`apps/web/src/app/features/admin/pages/admin-commission/`). That one is an append-only
*history* table because past order lines must resolve the rate that applied at purchase
time. A notification address has no such requirement — see Open Question 3.

`AdminController` (`apps/api/src/admin/admin.controller.ts`) is read-mostly today:
`users`, `orders`, `dashboard`, `analytics`, `earnings`, `audit-logs`, plus two user
`PATCH`es. There is no settings route.

Admin portal pages live at `apps/web/src/app/features/admin/pages/<name>/` — a
`admin-settings` page would follow `admin-commission`'s shape exactly.

### Vendor entity — no notification field

`apps/api/src/vendors/entities/vendor.entity.ts` has **no** notification-email-ish column.
The only address reachable is the owning account's: `Vendor.user` / `Vendor.userId`
(`:68-73`) → `User.email`.

**Edge case that matters:** `userId` is `nullable: true` (`:72-73`). Admin-created vendors
(`POST /vendors/admin`, `apps/api/src/vendors/vendors.controller.ts:47`) can exist with no
user account at all — for those, there is no account email to fall back to.

Vendor self-service update path already exists and is the natural home for the field:
`PATCH /vendors/:id` (`vendors.controller.ts:126-128`, `@Roles(VENDOR, ADMIN)`) →
`VendorsService.update` (`vendors.service.ts:247-264`, `Object.assign(vendor, updateDto)`)
→ `UpdateVendorDto` (`apps/api/src/vendors/dto/update-vendor.dto.ts`) →
`UpdateVendorRequest` (`libs/shared/src/contracts/vendor.ts:52`). Vendor portal screen:
`apps/web/src/app/features/vendor/pages/vendor-profile/`.

## Business rules this must honour

### 1. Notification is a side-effect, never a critical path

Sending mail must never fail, block, delay, or roll back an order or payment operation.
This follows the `SearchIndexerService` precedent quoted above.

**The current code does NOT satisfy this, and there is a latent bug to fix:**

`MailService.send` (`mail.service.ts:47-52`) handles the **returned** `error` object from
Resend but has **no try/catch**. If `client.emails.send()` *rejects* — DNS failure,
socket timeout, 5xx — the rejection propagates out of `send()`, out of
`sendPasswordReset` / `sendEmailVerification`, and out of the calling auth flow. Today
that means a 500 on `POST /auth/forgot-password`. It is survivable there.

It is **not** survivable in the order path. `orders.service.ts:264` awaits
`capturePayment` *after* the stock transaction has already committed (`:259`) — stock
decremented, cart cleared (`:256`). A throw at that point returns a 500 to a customer
whose order exists, whose stock is gone, and whose payment is captured. Putting a bare
`await mailService.send(...)` inside `capturePayment` would extend that failure window to
an external mail provider.

**Therefore, mandatory:**
- `MailService.send` gets a try/catch so a transport rejection can never escape.
- The order-paid notification is dispatched via an **emitted domain event**, not a direct
  awaited call inside `capturePayment`. The listener swallows and logs its own failures.

### 2. Vendors see only their own lines

[[Order State Machine]] is explicit: vendors "cannot read/act on other vendors' lines or
any platform-fulfilled lines", and `OrdersService.findAllForVendor` (`:110-132`) enforces
it via `order_items.vendorId`. A vendor notification email must therefore be **per-vendor**
and contain **only that vendor's lines**. It must not show the order total, other vendors'
products, or the customer's full basket.

One order can span multiple vendors plus platform lines (`vendorId` null — see
[[Listing Types & Vendor Rules]]). So one paid order can produce N vendor emails
(one per distinct vendor) plus exactly one platform email. A platform-only order produces
zero vendor emails and one platform email.

### 3. Money and currency rendering

Per [[Money & Currency Rules]] and the one-order-one-currency rule enforced at
`orders.service.ts:225-229`, every amount in an email renders with its explicit currency
(`order.currency`). ZAR and NAD are never summed and the 1:1 peg is never assumed in copy.

Vendor emails should show **gross line values only**, matching the existing
`VendorOrderLineDto` shape (`unitPrice`, `currency`, `quantity`). They must **not**
restate commission, net earnings, or payout amounts — [[Vendor Earnings & Commission]]
owns that accounting, with its own 48h damage-claim eligibility window and settlement
anchoring. An email that quotes a payout figure at order time would be wrong.

### 4. HTML escaping in templates (new, security-relevant)

Today's two emails interpolate only URL-encoded tokens (`mail.service.ts:20,31`), so
injection is not currently possible. Order emails interpolate **vendor-supplied free
text** — `order_items.productName`, `Vendor.businessName`, and the customer's shipping
`recipientName`. All interpolated values must be HTML-escaped by the template renderer
by default.

### 5. Unconfigured-mail no-op is preserved

The `RESEND_API_KEY`-absent warn-and-skip behaviour (`:42-45`) stays. Tests never send
real email — mock the transport.

## Scope

**In scope**

1. A reusable branded HTML email layout (placeholder logo, `docs/design/DESIGN.md`
   tokens — `--hb-primary` `#015300`, `--hb-on-primary` `#ffffff`), plus a plain-text
   fallback, with the two existing emails migrated onto it.
2. A `platform_settings` store + admin API + admin portal field for the platform
   notification address.
3. A `notificationEmail` override on `Vendor` + vendor portal field, defaulting to the
   vendor's account email.
4. An `order.paid` domain event emitted at `orders.service.ts:346-351`, with a listener
   that sends per-vendor and platform notification emails.
5. The `MailService.send` try/catch hardening (rule 1).

**Out of scope**

- **Customer order-confirmation email** — a real gap, but not what was asked for. See
  Open Question 1; deliberately not carded.
- Vendor approval/rejection, onboarding, shipment, delivery, cancellation or refund
  emails. (The template built here makes them cheap later.)
- A durable queue / retry / dead-letter store. In-process best-effort matches the
  existing `SearchIndexerService` precedent. See Open Question 2.
- Per-user notification preferences or unsubscribe management (transactional mail only).
- Replacing Resend, or adding a second provider.
- Marketing/bulk email, digests, or scheduled summaries.
- Email localisation / multi-language.
- Changing the order state machine, payment provider port, or any money math.

## `@hb/shared` contract impact

- **Template infra:** none. Rendering is entirely API-internal — no interface crosses the
  API/UI boundary.
- **Platform settings:** new `libs/shared/src/contracts/settings.ts` with
  `PlatformSettingsDto` + `UpdatePlatformSettingsRequest`, exported from
  `libs/shared/src/contracts/index.ts`. The admin DTO `implements` the request interface.
- **Vendor notification email:** `notificationEmail?: string` added to `VendorSelfDto`
  (`libs/shared/src/contracts/vendor.ts:33`) and `UpdateVendorRequest` (`:52`).
  **It must NOT go on `VendorDto` (`:11`)** — that interface is what the public
  storefront vendor profile renders, and publishing a vendor's contact address to
  anonymous visitors is a privacy regression. `AdminVendorDto` may optionally expose it.
- **Order-paid notifications:** none. Server-side only.

## Migrations

Both new migrations need a timestamp later than the current latest,
`1784678400000-WishlistItems.ts`.

- `platform_settings` table. Per `apps/api/CLAUDE.md`, any new non-nullable column needs a
  `DEFAULT` or a same-migration backfill so `migration:run` succeeds against an existing
  database.
- `vendors.notificationEmail` — nullable `varchar`, so no default or backfill needed;
  `NULL` means "use the account email".

## Implementation Notes

**Status: TE-1, TE-2, TE-3 built and merged (2026-08-15). TE-4 outstanding.**

**Branch:** `feat/e7WlyfLC-transactional-email-foundations` (6 commits, PR not yet opened)  
**Commits:**
- `e2ceb07` feat(api): branded email template, multi-recipient send, transport hardening
- `631b8f2` feat(api): platform notification email setting with audit-logged admin API
- `1082cd3` feat(api): vendor notification email override with account-email fallback
- `703f802` feat(web): admin settings page for platform notification recipients
- `cab39a6` feat(web): vendor notification email field with account-email fallback
- `2aa2348` fix(api): contain render failures inside the mail transport guard

**TE-1: Template API & multi-recipient dispatch**

New file: `apps/api/src/mail/email-template.ts`. Exports:
- `EMAIL_LOGO_MARKUP: string` — single reference point for logo swap (text wordmark on `#015300`).
- `EmailContentBlock` type: union of `{ type: 'heading'|'paragraph'|'link'; text: string }` and `{ type: 'rawHtml'; html: string; text: string }`. The `text` field on `rawHtml` is mandatory and separate because raw markup cannot be downconverted to plain text safely.
- `RenderedEmail { html: string; text: string }` — pair returned by `renderEmail()`.
- `renderEmail(subject: string, blocks: EmailContentBlock[]): RenderedEmail` — public API. Heading/paragraph/link content is HTML-escaped by default; `rawHtml` is the sole escape hatch, demanding its own plain-text paired field.

No template engine dependency added. Rendering is plain TypeScript with inline CSS (~600px), design tokens baked as hex literals (most mail clients strip CSS custom properties).

**Future callers:** Compose `EmailContentBlock[]`, pass to `renderEmail()`, dispatch result.

**`MailService.send()` signature:** `private send(to: string | string[], subject: string, blocks: EmailContentBlock[])` — takes **content blocks, not rendered HTML**. Deliberate: rendering happens inside the try/catch so a template error cannot sidestep the safety guarantee.

**Multi-recipient support:** `to` is now `string | string[]`. Resend dispatches a single batch (cap 50, matching DTO limit in TE-2). All recipients receive identical content.

**Plain-text part:** Every email carries a `text` alternative, auto-derived by stripping HTML tags. For emails with raw HTML, the `text` field on the block is used as-is.

**Business Rule 1 fix:** Rendering, client resolution, and transport dispatch all sit inside one try/catch. Template error, provider rejection, or DNS timeout are logged and swallowed. No-op-when-unconfigured (warn + return) is preserved.

**HTML escaping hardening:** `escapeHtml()` coerces nullish values to empty string because `apps/api` runs with `strictNullChecks: false`. TE-4 will pass optional fields straight into blocks, and escaping must not throw.

**Link href validation:** Restricted to `http(s)` scheme. Quote-escaping in the HTML attribute stops `"` breakout but not `javascript:` or `data:` injection, so scheme-check is mandatory.

**TE-2: Platform notification email setting**

**Database:** Migration `1786752000000-PlatformSettings.ts`. Table schema:
```
platform_settings(id uuid PK, notificationEmails jsonb NOT NULL DEFAULT '[]', updatedAt timestamptz DEFAULT now(), updatedByUserId uuid NULL)
```

`jsonb` chosen to align with repo precedent (`vendors.profileSections`). Singleton row seeded in `up()`.

**Shared contract:** `libs/shared/src/contracts/settings.ts` — `PlatformSettingsDto { notificationEmails: string[] }` and `UpdatePlatformSettingsRequest { notificationEmails: string[] }`, exported from index.

**API endpoints:** New `SettingsController` in `apps/api/src/settings/` (separate module, mirrors `commission` pattern).
- `GET /api/admin/settings` → `PlatformSettingsDto`
- `PATCH /api/admin/settings` → takes `UpdatePlatformSettingsRequest`, replaces entire list

Validation: `@IsArray()` + `@ArrayMaxSize(50)` + `@IsEmail({}, { each: true })`. Class-level `@Roles(UserRole.ADMIN)`.

**Audit trail:** Updates write `AuditAction.PLATFORM_SETTINGS_UPDATED` via `AuditService`, recording recipient **count** (not addresses — privacy).

**No-entry guard:** `get()` returns `{ notificationEmails: [] }` if row missing — never throws, so TE-4 can warn-and-skip.

**Web:** Admin portal page `apps/web/src/app/features/admin/pages/admin-settings/`, route `/admin/settings`. UI is a list editor (add/remove addresses).

**Q3 resolved (multi-recipient):** Yes. Platform email sent to all configured addresses in one batch.

**TE-3: Vendor notification email override**

**Database:** Migration `1786838400000-VendorNotificationEmail.ts`. Adds nullable `notificationEmail varchar` to `vendors`; no backfill.

**Shared contract:** `notificationEmail?: string | null` on `VendorSelfDto` and `UpdateVendorRequest` **only** — NOT on `VendorDto` (storefront-facing shape; publishing contact addresses to anonymous visitors is a privacy regression).

**Resolution order:** `VendorsService.resolveNotificationEmail(vendorId): Promise<string | null>`:
1. If override set → use it.
2. Else if `vendor.user.email` exists → use account email.
3. Else `null` (handles admin-created vendors with no user).

Throws `NotFoundException` only if vendor id doesn't exist.

**Clearing:** Send `null` or `''` to clear; send nothing (`undefined`) means unchanged.

**Q5 resolved (format validation only):** `@IsEmail` only, no verification round trip. Vendor-owned config (not auth identity). Trade-off: typo'd addresses silently black-hole notifications.

**Web:** Vendor portal field in `apps/web/src/app/features/vendor/pages/vendor-profile/`.

**Resolved open questions:**
- **Q3:** Multiple recipients → yes, array in platform settings + dispatch + vendor override.
- **Q4:** Single-row table, typed columns, audit-logged via `AuditService`.
- **Q5:** Format validation only (vendor-owned operational config).
- **Q6:** Text wordmark in `EMAIL_LOGO_MARKUP` constant, swappable in one place.
- **Q7:** No change to `MAIL_FROM` per type (kept single sender).

**Test results:** API 670/670 tests ✓, Web 818/818 tests ✓. Migrations run → revert → re-run cleanly against dev db.

**TE-4 outstanding:** `MailService.send()` remains `private`. TE-4's listener needs a public method (e.g., `sendOrderPaidVendor()`) or visibility widened. Record for TE-4 planning — single loose thread.

---

## Open questions (a human should answer before `/ship-card`)

1. **Should the customer get an order-confirmation email?** Today they get nothing after
   paying. This was not in the request, so no card exists for it — but it is the most
   visible gap the research turned up, and the machinery built by TE-4 makes it a small
   follow-up. Confirm whether to add it.
2. **Retry / queue policy.** Recommendation: no retry, no queue, in-process best-effort —
   consistent with `SearchIndexerService`, whose safety net is a daily reindex. Note that
   notifications have **no** equivalent safety net: a lost order email is lost silently
   (a warning in the logs is the only trace). Accept that for v1, or is a durable
   outbox/retry required?
3. **Platform notification address: one recipient or several?** `MailService.send`
   accepts a single `to` string today. If ops wants finance + fulfilment + support on it,
   the column should be an array/CSV and `send()` must accept multiple recipients. This
   materially changes the settings schema — decide before TE-2 is built.
4. **Settings storage shape.** Recommendation: a single-row `platform_settings` table with
   typed columns (not a generic key/value bag, not the append-only history pattern
   `commission_rates` uses — no historical resolution is needed here), with changes
   recorded through the existing `AuditService` since changing who receives payment
   notifications is a security-relevant config change. Confirm.
5. **Vendor override verification.** Should a vendor-supplied `notificationEmail` require
   a confirm-this-address round trip before it is used, or is validated-format-plus-save
   enough for v1? Recommendation: format validation only for v1 (it is vendor-owned
   operational config, not an auth identity), with a note that unverified addresses can
   silently black-hole notifications.
6. **Placeholder logo asset.** Is there an approved H&B logo file to use, or should the
   template ship a text/monogram placeholder? `docs/design/DESIGN.md` defines colour and
   type tokens but no logo asset. Recommendation: text wordmark on `--hb-primary` until
   an asset is supplied, referenced from one constant so swapping it is a one-line change.
7. **`MAIL_FROM` per email type.** All mail currently sends from `no-reply@hb-ecommerce.com`
   ([[Auth & Roles]] recorded this as the agreed sender). Should operational notifications
   use a different sender or a `reply-to` so a vendor can reply to a human? Not currently
   supported by `send()`.

## Slices → Trello

Three independent slices, then one integration slice that depends on all three.

| Card | Trello | Title | Depends on |
|---|---|---|---|
| TE-1 | [e7WlyfLC](https://trello.com/c/e7WlyfLC) | Branded reusable HTML email template + migrate existing emails onto it | — |
| TE-2 | [vFpGIEUF](https://trello.com/c/vFpGIEUF) | Platform notification email setting — `platform_settings` entity, migration, admin API + UI | — |
| TE-3 | [0DwUaRkX](https://trello.com/c/0DwUaRkX) | Vendor-configurable notification email — vendor portal override with account-email default | — |
| TE-4 | [wygImWJb](https://trello.com/c/wygImWJb) | `order.paid` domain event → vendor + platform order-confirmed notification emails | TE-1, TE-2, TE-3 |

TE-1/TE-2/TE-3 touch disjoint files and can run in parallel. TE-4 is the only slice that
touches order/payment logic and therefore the only one carrying a mandatory
money/order-state unit-test obligation.

**Cross-slice coordination:** TE-2 and TE-3 each add a migration. Both must be timestamped
later than the current latest (`1784678400000-WishlistItems.ts`) and must not collide with
each other if they land close together.

TE-1 hardens `MailService.send` with a try/catch, and TE-4 depends on that hardening — so
TE-1 must merge before TE-4 regardless of the order of the other two.
