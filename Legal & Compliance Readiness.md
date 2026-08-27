# Legal & Compliance Readiness

Front-of-funnel spec. Implementation flows through `/ship-card` — no code here.
Related: [[HB Domain Model]] · [[Cross-Border & Customs]] · [[Money & Currency Rules]] ·
[[Listing Types & Vendor Rules]] · [[Vendor Earnings & Commission]] · [[Auth & Roles]] ·
[[Landing Site Migration]] · [[Configurable Shipping Fee]].

## Problem

The site collects customer PII (accounts, orders, addresses) and runs Google Analytics
behind an existing consent gate (`apps/web/src/app/core/consent/`), but **no Privacy
Policy, Cookie Policy, Terms of Service/Trade, Shipping Policy, Returns/Refund Policy,
or Export & Customs page exists anywhere in the codebase.** Confirmed by search — zero
matches for any of these beyond the consent banner itself.

Two concrete gaps make this urgent, not hypothetical:

1. **The consent banner already promises a policy that doesn't exist.**
   `consent-banner.html:6` says "See our POPIA notice for details" — that notice is not
   built. The banner is also analytics-only accept/decline (no granular categories),
   which is the right shape for a first cut but the copy is making a promise the site
   can't keep yet.
2. **The Terms/Privacy links at signup are dead.** [[Auth & Roles]]'s login/register
   implementation note records: *"the terms-acceptance checkbox and 'remember me' were
   dropped (absent from the Stitch designs)"* and *"Terms/Privacy links render as
   'coming soon' snackbars — they make no API calls."* There is currently **no consent
   capture at signup at all** — not even a checkbox, let alone a logged, timestamped
   acceptance record.

Timing: [[Landing Site Migration]] records Marketplace v1 targeting **1 October 2026**
(H&B Brain `18-product-roadmap.md`, confirmed 2026-08-13) — roughly five weeks from
today (2026-08-24). This is genuinely launch-blocking, not backlog.

## Which legal regime actually governs this (researched — not in the vault, verify with counsel)

The vault does not record HB's registered legal entity, registration number, or
registered address anywhere I could find (searched explicitly — no hits). The user's
framing assumes a Namibia-registered company; the research below is written against
that assumption but the entity facts still need confirming — see Open Questions.

**Namibia has no data protection law in force yet.** A Data Protection Bill (drafted
2022) was reported in January 2026 as fast-tracked for cabinet resubmission, but it has
not been passed by parliament or commenced. There is nothing to comply with today on
the Namibian data-protection side — but the bill exists and should be monitored, and
writing the Privacy Policy in POPIA's shape (below) means it will already read
compatibly with a future Namibian Act, which tends to follow the same
GDPR-family structure.

**POPIA (South Africa) is the real operative regime regardless of where HB is
incorporated**, for two independent reasons:
- POPIA has an extraterritorial "means" test: it applies to a responsible party not
  domiciled in South Africa if it uses automated or non-automated **means in South
  Africa** to process personal information — South African servers, vendors, payment
  processors, or a local sourcing/fulfilment operation all cross this line. HB's
  Procurement Service sources from South African-facing retailers and its vendor base
  is SA-facing; the shortlisted payment processors (Stitch, DPO Group) are South
  African-licensed. This alone is very likely to bring HB's processing under POPIA.
- **Some customers are SA-domiciled data subjects in their own right.** [[Configurable
  Shipping Fee]] and [[Order State Machine]] both treat a `ZA→ZA` domestic order as a
  real case (`orders.destinationCountry` can legitimately be `ZA`) — HB is not
  Namibia-only. South African customers' data is South African personal information
  under POPIA on its own terms, independent of the extraterritoriality question above.

**Namibia's Electronic Transactions Act 4 of 2019** (in force since 16 March 2020) *is*
real, current Namibian law and applies to any e-commerce transaction reaching a
Namibian consumer, including from a foreign supplier — its consumer-protection chapter
is stated to apply to cross-border transactions with foreign entities. Confirmed
provisions relevant here:
- Marketing sent as an electronic "data message" must disclose the sender's identity,
  contact details and place of business, provide a working opt-out, and state where the
  sender obtained the recipient's personal information; unsolicited commercial messages
  require **opt-in**, not opt-out.
- Suppliers must disclose accurate goods/services information, price, delivery
  timelines and terms before the sale.
- A **7-day cooling-off period** lets a consumer cancel; if the supplier fails the
  information-disclosure duties above, the consumer can cancel and get a **full refund
  including the cost of returning the goods**.

**South Africa's CPA + ECTA** converge on the same shape for the ZA-domiciled slice of
customers: mandatory pre-sale supplier disclosure, a right to cancel within 7 days for
goods bought without face-to-face contact.

**Net effect for drafting:** POPIA supplies the privacy/data-subject-rights template.
Namibia's ETA and South Africa's CPA/ECTA independently converge on the same 7-day
cooling-off + pre-sale disclosure shape, so **one** Returns/Refunds Policy and **one**
mandatory-disclosure block on Terms of Trade satisfies both without drafting two
versions. None of this is a substitute for a qualified attorney's sign-off, particularly
on: which country's law should govern the contract (Open Questions Q1), whether the
Namibian ETA's cross-border consumer-protection reach actually binds a South
African-incorporated entity if that's what HB turns out to be (Open Questions Q2), and
whether an Information Officer must be registered with South Africa's Information
Regulator (near-certainly yes if POPIA applies — Open Questions Q3).

Sources consulted (WebSearch, 2026-08-24): DLA Piper Data Protection Laws of the World
(Namibia), APC.org reporting on the 2022 draft bill's 2026 fast-tracking, ITLawCo and
NamibLII on the Electronic Transactions Act 4 of 2019, ENSafrica/Mondaq on POPIA's
extraterritorial scope.

## Business facts this must stay consistent with (from this vault — do not restate differently)

- **Brand name: "H&B E-Commerce"** ([[Landing Site Migration]] Decision 3 — already
  swept across the app). Legal-doc templates below use this consistently; do **not**
  reintroduce "H&B Market"/"H&B Marketplace".
- **Contact facts already public** ([[Landing Site Migration]] `/contact` page,
  shipped): `+264 81 355 9921`, `info@hb-ecommerce.com`, WhatsApp
  `wa.me/264813559921`, support **acknowledgement within 1 business day**, English and
  Afrikaans, damage claims within **48 hours of delivery**. Any legal-doc SLA language
  must match these exactly — this is the same rule [[Landing Site Migration]] already
  enforced on the Contact page.
- **Dual-engine business model** ([[HB Domain Model]], [[Landing Site Migration]]):
  Engine 1 "Procurement Service" — HB buys on the customer's behalf from
  Takealot/Amazon/Shein/Temu/individual vendors, briefly owns the goods, resells at a
  tiered margin (20% under R500, 15% R500–1,000, 10% over R1,000), shipping + customs
  passed through. Engine 2 "Marketplace" — vendors sell their own goods, **H&B never
  owns marketplace inventory**. This distinction matters legally, not just in
  marketing: HB is the **seller of record** on Procurement Service transactions and a
  mere **intermediary/marketplace operator** on vendor transactions, and liability,
  returns handling, and customs paperwork differ by engine. Terms of Trade and the
  Returns Policy must draw this line explicitly.
- **SACU: no customs duty on South African-origin goods entering Namibia** ([[Cross-Border
  & Customs]]) — ZA and NA share a customs union. **But this does NOT cover the
  Procurement Service's non-SACU sourcing** (Amazon/Shein/Temu are not South African
  origin) — those are genuine international imports into Namibia and *do* attract
  Namibian import duty/VAT. The Export & Customs page must not blanket-state "no duty" —
  that's only true for the Marketplace engine and SA-origin Procurement purchases.
- **No payment gateway is live** ([[Money & Currency Rules]]) — shortlist is Stitch
  (Namibia support looks unlikely), FNB Namibia eCommerce Switch, DPO Group (licensed
  Namibian facilitator); none confirmed. `PAYMENT_PROVIDER` is a logging stub. Legal
  docs must not name a specific processor or claim a specific payment security
  certification (e.g. PCI-DSS) until a real provider is chosen — see Open Questions Q4.
- **No refund flow exists in code** ([[Vendor Earnings & Commission]] confirms:
  `payment-status.ts`'s `refunded` state exists but nothing currently writes it).
  The Returns/Refunds Policy can and must still make the legal commitment (7-day
  cooling-off, full refund) — that's a legal obligation independent of whether the
  system automates it yet — but must not claim an automated self-service refund flow
  that doesn't exist. State refunds are processed manually by support for now.
- **Vendor KYC/document upload is deferred past v1** ([[Listing Types & Vendor Rules]])
  — the vendor onboarding fields (`businessName`, `tradingName?`, `registrationNumber?`,
  `website?`, `description?`, `countryCode?`) are self-declared, not verified. The
  Vendor/Marketplace Agreement must not promise HB has verified a vendor's legal
  identity or trading right — it hasn't, by design, at v1.
- **Vendor commission: 15%, vendor keeps 85%**, admin-configurable and effective-dated
  ([[Vendor Earnings & Commission]]). Public-facing vendor agreement language should
  state the current rate but flag it can change (matching the actual `commission_rates`
  history mechanism), not claim it's fixed.
- **ZAR/NAD 1:1 peg is data, never assumed** ([[Money & Currency Rules]]) — legal docs
  should describe both currencies existing, not describe an FX conversion (there isn't
  one).

## Scope

In scope — draft the following as real, ship-ready copy (not lorem-ipsum placeholders),
each flagged inline wherever a company-specific fact or a genuine legal judgment call is
needed:

1. Privacy Policy
2. Cookie Policy
3. Terms of Service / Terms of Trade (includes mandatory ETA/CPA pre-sale disclosure block)
4. Shipping Policy
5. Returns & Refunds Policy
6. Export & Customs Terms
7. Vendor/Marketplace Agreement (Terms of Trade for vendors, distinct from customer ToS)
8. Consent-capture wiring: a real checkbox at signup (currently absent), a real Cookie
   Policy for the existing consent banner to link to, and a logged/timestamped
   acceptance record.

Out of scope for this spec (see "Out of scope" below): PAIA manual drafting itself
(flagged as required, but the manual's content depends on facts — data flows, retention
schedules — this note doesn't have yet), actual Information Officer registration
(an administrative filing, not a code or copy task), payment-processor-specific
compliance language (blocked on Open Questions Q4).

## Templates

Every template below is real, usable draft copy. **Bracketed items in
`[ALL-CAPS-WITH-BRACKETS]`** are facts only a human can supply (legal entity name,
registration number, registered address, Information Officer contact) — implementation
cards must not invent these. **Paragraphs marked `⚠ NEEDS LEGAL REVIEW`** are judgment
calls (liability caps, governing law, indemnity, arbitration) that a qualified attorney
should confirm or rewrite before publishing, not just proofread.

---

### 1. Privacy Policy

```markdown
# Privacy Policy

Last updated: [DATE]

H&B E-Commerce ("H&B", "we", "us") operates hb-ecommerce.com and the H&B mobile and
web storefront (the "Platform"). This policy explains what personal information we
collect, why, and what rights you have over it.

## Who we are

[LEGAL ENTITY NAME], registration number [REGISTRATION NUMBER], registered address
[REGISTERED ADDRESS]. Our appointed Information Officer is [INFORMATION OFFICER NAME],
contactable at [INFORMATION OFFICER EMAIL].

## What we collect

- **Account information**: name, email, phone number, password (stored hashed, never
  in plain text).
- **Order information**: delivery address, billing details, order history, items
  purchased, order value.
- **Vendor information** (if you register as a vendor): business name, trading name,
  registration number, website, country, and (optional, not required at this time)
  a verification document.
- **Usage information**: pages visited, device/browser type, and analytics events —
  only if you've consented via our cookie banner (see our [Cookie Policy]).
- We do **not** collect or store payment card numbers. Payments are handled by
  [PAYMENT PROVIDER — TBD, see Open Questions] and any card data goes directly to
  them, not through H&B's servers.

## Why we process it (lawful basis)

- To create and run your account, and to fulfil orders you place — contractual
  necessity.
- To communicate with you about orders, verification, and password resets —
  contractual necessity / legitimate interest.
- To run analytics that help us improve the Platform — your consent, which you can
  withdraw at any time (see Cookie Policy).
- To meet legal obligations (e.g. tax and consumer-protection recordkeeping).

## Who we share it with

- **Vendors**, for the specific order lines they need to fulfil (name, delivery
  address, items ordered) — never your full account or payment details.
- **Delivery/logistics partners**, to get your order to you across the SA↔Namibia
  corridor.
- **[PAYMENT PROVIDER — TBD]**, to process payment — they act as an independent
  responsible party for the payment data they hold.
- **Google Analytics**, only if you've consented to analytics cookies.
- We do not sell your personal information to anyone.

## Where it's processed

Our servers and much of our vendor/logistics network are based in South Africa; if you
are in Namibia, this means your information crosses the border for processing. We take
reasonable steps to protect it in transit and at rest regardless of which side of the
border it's on.

## How long we keep it

[RETENTION SCHEDULE — TBD, see Open Questions. Suggested default pending a real
records-retention decision: account data for the life of the account plus a limited
period after closure for legal/tax purposes; order records for the period required by
tax law in the relevant jurisdiction.]

## Your rights

You can ask us to:
- Confirm what personal information we hold about you and get a copy of it.
- Correct information that's wrong or incomplete.
- Delete your information, subject to our legal obligation to keep certain records
  (e.g. completed order/tax records).
- Object to processing, including analytics — withdraw cookie consent at any time.

To exercise any of these, email [INFORMATION OFFICER EMAIL]. We'll respond within a
reasonable time and in any event within the timeframe the law requires.

## Complaints

If you're unhappy with how we've handled your information, you can complain to us
first at [INFORMATION OFFICER EMAIL], or directly to South Africa's Information
Regulator (enquiries@inforegulator.org.za) [⚠ NEEDS LEGAL REVIEW — confirm whether a
Namibian regulator contact should also be listed once Namibia's Data Protection Act
is in force; today there is no equivalent Namibian body to name].

## Changes to this policy

We'll post any changes here and update the "Last updated" date. Material changes will
also be flagged in-app.
```

---

### 2. Cookie Policy

```markdown
# Cookie Policy

Last updated: [DATE]

This page explains the cookies H&B E-Commerce uses and how to control them. It's the
policy referenced by the cookie banner you see when you first visit the Platform.

## What are cookies

Small files stored on your device that let a website remember information between
visits.

## What we use

| Category | Purpose | Can you decline? |
|---|---|---|
| **Strictly necessary** | Keep you logged in (session/refresh tokens), remember your cart, basic security (CSRF protection) | No — required for the site to function |
| **Analytics** | Google Analytics — understand how the storefront is used, so we can improve it | Yes — declined by default until you accept |

We currently run one consent-gated analytics integration (Google Analytics) plus a
first-party, non-cookie usage-events feature used for internal product analytics that
is not gated by this banner because it does not use cookies or track you across sites.
[⚠ NEEDS LEGAL REVIEW / VERIFY: confirm the first-party analytics feature genuinely
sets no cookies and collects no data that would itself require consent under POPIA/the
ETA before publishing this claim — check the current `AnalyticsService` implementation
at ship time, not just at spec time.]

## Your choice

When you first visit, a banner lets you **Accept** or **Decline** analytics cookies.
Your choice is stored on your device and applies until you change it. You can change
your choice at any time via [Cookie preferences link/control — TBD, see Open
Questions: does this need a persistent "manage cookies" control somewhere other than
the first-visit banner, e.g. a footer link, since the banner itself disappears once a
choice is made?].

## Third-party cookies

Google Analytics, when accepted, sets its own cookies under Google's privacy policy
(https://policies.google.com/privacy). We don't control what Google does with
analytics data beyond what our GA configuration sends it.
```

---

### 3. Terms of Service / Terms of Trade

```markdown
# Terms of Service

Last updated: [DATE]

These terms govern your use of the H&B E-Commerce platform (hb-ecommerce.com) and any
purchase you make through it. By creating an account or placing an order, you agree to
them.

## 1. Who we are (mandatory pre-sale disclosure)

[LEGAL ENTITY NAME], registration number [REGISTRATION NUMBER], registered address
[REGISTERED ADDRESS]. Contact: info@hb-ecommerce.com, +264 81 355 9921, WhatsApp
wa.me/264813559921. Support responds within 1 business day; English and Afrikaans.

[This section exists to satisfy Namibia's Electronic Transactions Act 4 of 2019 and
South Africa's Consumer Protection Act / ECTA, both of which require suppliers to
disclose their identity, contact details, and place of business before a consumer
transacts online. ⚠ NEEDS LEGAL REVIEW: confirm this discharges the disclosure duty in
full — the Act's exact wording may require additional fields (e.g. VAT number).]

## 2. Two kinds of purchase on this Platform

H&B runs two distinct services and your rights differ slightly depending on which one
you're using:

- **Marketplace purchases** — you're buying directly from an independent vendor.
  H&B is the platform operator, not the seller. The vendor is responsible for the
  accuracy of their listing and for fulfilling the order; H&B facilitates payment,
  cross-border logistics, and dispute support.
- **Procurement Service purchases** — H&B buys the item on your behalf from a
  third-party retailer (e.g. Takealot, Amazon, Shein, Temu) or an individual seller,
  takes brief ownership of it, and resells it to you at a service margin (20% on
  items under R500, 15% on R500–R1,000, 10% over R1,000; shipping and customs costs
  are passed through at cost). For these purchases, **H&B is the seller of record**.

Your order will show which kind each item is.

## 3. Placing an order

- Prices are shown in the currency of the item's country of origin (ZAR or NAD).
  Namibian Dollar is pegged 1:1 to the South African Rand; we do not perform currency
  conversion — each price is set explicitly in its own currency.
- An order confirmation does not guarantee stock; if an item becomes unavailable
  after you order, we'll tell you and refund that line.
- All prices shown are final and include our margin/commission where applicable —
  no hidden fees are added at checkout beyond the shipping fee (see [Shipping Policy])
  shown before you confirm payment.

## 4. Payment

[⚠ NEEDS LEGAL REVIEW / TBD — no payment provider is confirmed yet (see Open
Questions Q4). Do not publish payment-method-specific language (e.g. "we accept card
payments via X") until a provider is chosen and its terms are known. Placeholder:]
"Payment is processed by a licensed third-party payment provider. We do not store your
card details."

## 5. Delivery

See our [Shipping Policy] for timeframes, costs, and who bears risk in transit.

## 6. Your right to cancel (cooling-off)

You may cancel your order within **7 days** of placing it, for any reason, in line
with Namibia's Electronic Transactions Act and South Africa's Consumer Protection Act.
See our [Returns & Refunds Policy] for the full process. If we fail to give you the
information this section 1 promises before you order, you may cancel and receive a
full refund including the cost of returning any goods already delivered.

## 7. Liability

⚠ NEEDS LEGAL REVIEW — do not publish a liability-limitation clause without counsel.
Placeholder structure only: H&B's liability for Marketplace purchases is limited to
facilitating the transaction and resolving disputes in good faith between you and the
vendor; H&B's liability for Procurement Service purchases, where H&B is the seller of
record, is [governed by ordinary consumer-protection law / a stated cap — TBD]. Nothing
in these terms excludes liability that cannot lawfully be excluded.

## 8. Governing law

⚠ NEEDS LEGAL REVIEW / BLOCKING — see Open Questions Q1. This section cannot be
filled in until the legal entity and its registered jurisdiction are confirmed.

## 9. Changes to these terms

We may update these terms; material changes will be flagged in-app and by email to
account holders.
```

---

### 4. Shipping Policy

```markdown
# Shipping Policy

Last updated: [DATE]

## Where we deliver

Anywhere in Namibia, and domestically within South Africa. Delivery beyond Windhoek is
available at [your own higher cost / a stated additional fee — TBD, matches H&B Brain
`02-vision.md` / `06-target-customers.md`'s "beyond-Windhoek at the customer's own
higher cost" framing; the exact fee schedule is separate from this policy's job, see
[[Configurable Shipping Fee]]].

## Shipping fee

A shipping fee is shown as a separate line item at checkout before you pay, and is
part of your order total. [Exact amount(s) — TBD, blocked on [[Configurable Shipping
Fee]] Open Question 1/2, which are themselves blocking implementation. Do not publish
a number here until that spec's Q1/Q2 are resolved.]

## How long delivery takes

[TIMEFRAMES — TBD. [[Cross-Border & Customs]] lists "SLA per leg (domestic ZA, border,
domestic NA)" as an open question for a human to answer — this policy cannot state
real numbers until that's resolved. Do not publish estimated days without an ops
answer; an unmet published SLA is a real legal and reputational exposure, not just an
inconvenience.]

## Cross-border handling

Orders crossing from South Africa into Namibia pass through customs as part of normal
delivery — see our [Export & Customs Terms] for what that means for duties and
paperwork. South Africa and Namibia share a customs union (SACU), so goods of South
African origin generally cross without import duty; this does not apply to items
sourced by our Procurement Service from outside South Africa (see Export & Customs
Terms).

## Risk in transit

⚠ NEEDS LEGAL REVIEW: confirm who bears risk of loss/damage in transit — typically
either "risk passes to the customer on delivery" or "risk stays with the seller until
delivery," and this may legitimately differ between Marketplace and Procurement Service
purchases given who is seller of record. Not stated here pending that confirmation.

## Vendor-fulfilled orders

Marketplace orders are shipped from the vendor to H&B's logistics handoff, then
onward to you — see our Terms of Service for how Marketplace purchases work. This may
mean a marketplace order takes an extra step compared to a Procurement Service order;
[if this creates a materially different delivery timeframe, state it once real SLA
numbers exist].
```

---

### 5. Returns & Refunds Policy

```markdown
# Returns & Refunds Policy

Last updated: [DATE]

## Your right to change your mind (cooling-off)

You can cancel any order within **7 days** of placing it, for any reason, and get a
full refund. This right comes from Namibia's Electronic Transactions Act and South
Africa's Consumer Protection Act, both of which apply to online purchases made without
face-to-face contact.

To cancel, contact us at info@hb-ecommerce.com or WhatsApp wa.me/264813559921 — we
acknowledge within 1 business day.

## If something arrives damaged or wrong

Tell us within **48 hours of delivery** — this is our damage-claim window. Contact
support with your order number and, where possible, photos of the issue.

## How refunds work today

⚠ IMPORTANT — reflects actual current system state, not aspirational copy: refunds are
currently **processed manually by our support team**, not through an automated
self-service flow. This does not change your legal right to a refund — it means the
mechanics of getting your money back involve a human on our side rather than an instant
system reversal. [Once an automated refund flow exists in the product, update this
section to describe it — see Open Questions Q6 in [[Vendor Earnings & Commission]]'s
"no real refund flow exists in code" note, which this policy's promises now depend on.]

Refunds are issued to the original payment method, in the currency you paid in (ZAR or
NAD — we do not convert between them).

## Marketplace vs Procurement Service returns

- **Marketplace purchases**: the vendor is responsible for the item's condition and
  accuracy of description. We mediate the return between you and the vendor and hold
  the vendor's payout accordingly (see how vendor earnings work — [[Vendor Earnings
  & Commission]]) until the return is resolved.
- **Procurement Service purchases**: H&B is the seller of record, so returns are
  handled directly with us, subject to the original retailer's own return policy where
  the item was sourced from a third party (e.g. some marketplaces or individual sellers
  may not accept returns on our end once we've paid them) — [⚠ NEEDS LEGAL REVIEW: how
  H&B's own cooling-off obligation to the customer interacts with a non-returnable
  purchase H&B itself made from a third party is a real commercial risk, not just a
  copy question. Flag for counsel and for the business to price into the margin.]

## What doesn't qualify

[TBD — e.g. perishables, made-to-order/personalized items, items outside the 7-day
window without a damage claim. Only add exclusions the business actually intends to
enforce; do not invent a boilerplate list.]
```

---

### 6. Export & Customs Terms

```markdown
# Export & Customs Terms

Last updated: [DATE]

## The short version

- If you're buying a **Marketplace** item or a **Procurement Service** item sourced
  from within South Africa, it generally crosses into Namibia without import duty —
  South Africa and Namibia are both members of the Southern African Customs Union
  (SACU).
- If your **Procurement Service** item was sourced from outside South Africa (for
  example Amazon, Shein, or Temu), it is a genuine international import into Namibia
  and **may attract Namibian import duty and VAT**, which is not included in H&B's
  margin and is passed through to you at cost.

## Who's the exporter/importer of record

⚠ NEEDS LEGAL REVIEW / TBD — [[Cross-Border & Customs]] lists "customs documentation
requirements and who generates them" as an open question for a human to answer. This
section cannot be completed honestly until that's resolved. Do not publish a specific
claim about who is legally the exporter/importer of record until ops/legal confirms it.

## Restricted and prohibited goods

[TBD — list of goods H&B will not source or ship cross-border, e.g. items prohibited
under Namibian import law. This needs input from whoever owns customs compliance; do
not invent a list.]

## Tracking customs status

Shipments carry a customs reference once they reach the border. [[Cross-Border &
Customs]] confirms a shipment cannot leave the "at border" stage without a
`customsReference` — a real business rule already enforced in the order/shipment data
model. [Once shipment tracking is customer-visible, link it here; today it may not be
— verify against the actual product surface at ship time.]

## Which courier

[TBD — [[Cross-Border & Customs]] lists "which courier(s) for the ZA→NA leg" as an
open question. Do not name a courier here until one is chosen.]
```

---

### 7. Vendor / Marketplace Agreement

```markdown
# Vendor Agreement

Last updated: [DATE]

This agreement governs your participation as a vendor on the H&B E-Commerce
marketplace. By submitting a vendor application, you agree to it.

## 1. What you're agreeing to sell through

You sell your own goods directly to H&B customers through our Platform. **H&B never
takes ownership of your inventory** — you remain the seller of record for every item
you list. This is different from H&B's own "Procurement Service," which is a separate
service where H&B itself buys and resells.

## 2. Onboarding

At present, vendor onboarding is **self-declared, not independently verified**: you
provide your business name, (optional) trading name, (optional) registration number,
website, and country. H&B does not currently require or review supporting documents
(e.g. proof of registration) before approving a vendor account. [⚠ NEEDS LEGAL REVIEW:
confirm this is an acceptable risk position for the business — a vendor agreement that
doesn't require identity verification is a real fraud/liability exposure, not just a
product-scope note, and [[Listing Types & Vendor Rules]] confirms this is a deliberate
v1 decision, not an oversight, so this paragraph must stay accurate to that rather than
overstate what verification actually happens.] You confirm that everything you provide
is accurate and that you have the legal right to sell the goods you list.

## 3. Commission

H&B charges a commission on each sale, currently **15%** of the item's price — you
receive **85%**. This rate can change; we'll always apply the rate that was in effect
at the time of the specific sale, never retroactively change what you earned on a past
order. [Current rate confirmed 2026-07-27 per [[Vendor Earnings & Commission]] — verify
this is still the live rate before publishing, since it's admin-configurable.]

## 4. Getting paid

Your balance for a sale becomes eligible for payout once the order is marked
delivered and a **48-hour damage-claim window** has passed with no claim. Eligible
balances accrue weekly and are batched into a settlement roughly every two weeks.
[⚠ TBD — [[Vendor Earnings & Commission]] confirms no actual payout/settlement
execution exists yet; this section describes the accounting model, not a live payment
mechanism. Do not publish this section implying automatic bank payouts happen today —
confirm actual payout mechanics with ops before this agreement goes live to real
vendors, since a written promise here is a real commitment even if the code doesn't
enforce it yet.]

## 5. Cancellations

If an order is cancelled because of something on your side (wrong stock, refusal to
fulfil, etc.), a cancellation fee may apply. [Amount TBD — [[Vendor Earnings &
Commission]] confirms this is a genuinely open number upstream; do not invent one.]
Cancelled or refunded lines never count toward your earnings.

## 6. Fulfilment obligations

You must hand off confirmed orders to H&B's logistics within [TIMEFRAME — TBD] of
confirmation. Repeated failure to fulfil may result in suspension (see [[Listing Types
& Vendor Rules]] vendor lifecycle: `pending → approved`, `approved → suspended`).

## 7. Your listings

You're responsible for the accuracy of your product listings — description, price,
stock levels, images you have the right to use. H&B may remove a listing that violates
these terms or applicable law without notice in urgent cases, with notice otherwise.

## 8. Termination

Either party can end this agreement at any time; H&B may suspend or reject a vendor
per the approval lifecycle already in place. [⚠ NEEDS LEGAL REVIEW: notice periods,
handling of in-flight orders and pending payouts on termination — not specified here.]
```

---

## Consent-capture wiring (code changes this spec drives)

Not covered by the templates above but required for them to mean anything:

- **Signup consent checkbox** — currently absent (`Auth & Roles` implementation note
  confirms it was dropped). Add a required, non-pre-ticked checkbox on register: "I
  agree to the [Terms of Service] and [Privacy Policy]." Must be logged with a
  timestamp (new field or a lightweight audit-log entry via the existing
  `AuditService` — see [[Vendor & Admin Portals]]'s audit-log precedent — so consent
  can be proven later, not just inferred from account existence).
- **Terms/Privacy links, currently "coming soon" snackbars** on login/register — wire
  to real routes once the pages exist.
- **Cookie banner** (`consent-banner.html:6`) — its "See our POPIA notice for details"
  line should link to the real Privacy Policy page (or a dedicated Cookie Policy page —
  decide at card time). Currently links nowhere.
- **New static routes**: `/legal/privacy`, `/legal/cookies`, `/legal/terms`,
  `/legal/shipping`, `/legal/returns`, `/legal/customs`, `/legal/vendor-agreement` (or
  similar — naming TBD at card time), following the same `RenderMode.Prerender`
  pattern [[Landing Site Migration]] already established for `/about`/`/services`/
  `/contact` (static marketing-shaped pages).
- **Footer links** — [[Landing Site Migration]] LSM-6 deliberately left "Shipping
  Policy" and "Terms of Trade" as dead `href="#"` links, matching the precedent that
  genuinely-unbuilt pages stay dead rather than pointing at nothing. This spec is what
  unblocks wiring them for real.

## @hb/shared contract impact

- Possible new `contracts/legal.ts`: `AcceptTermsRequest` (or fold into
  `RegisterRequest` as a required boolean — decide at card time; the "never duplicate
  DTOs" rule favours extending the existing register contract if the timestamp can be
  derived server-side from request time).
- If consent is logged via `AuditService`, no new contract needed — reuse the existing
  `AuditLogDto` shape with a new `AuditAction` value (e.g. `'user.terms_accepted'`),
  matching the pattern [[Vendor & Admin Portals]]'s audit log already uses for other
  user-lifecycle events.

## Out of scope (recorded so nobody assumes them)

- **PAIA manual** — required under POPIA once an Information Officer is registered,
  but its *content* (data flows, categories of information held, retention schedules)
  depends on facts this note doesn't have. A follow-up card once the entity facts in
  Open Questions are answered.
- **Information Officer registration itself** — an administrative filing with South
  Africa's Information Regulator, not a code or copy task. Human action, not a card.
- **Payment-processor-specific compliance language** (e.g. PCI-DSS statements) —
  blocked on Open Questions Q4 (no processor chosen).
- **A working automated refund flow** — the Returns Policy commits to the legal right;
  building the actual refund execution mechanism is separate, larger work already
  flagged as out of scope in [[Vendor Earnings & Commission]].
- **Restricted/prohibited-goods list** — needs ops/compliance input this note doesn't
  have; flagged in the Export & Customs template, not resolved here.
- **Namibian Data Protection Act compliance** — not yet enacted; nothing to build
  against today. Revisit if/when it commences.

## Open questions (a human must answer before publishing any of these pages)

1. **Legal entity facts — blocking everything.** Legal entity name, registration
   number, registered address, and which jurisdiction (Namibia or South Africa, or
   both if there are two entities) actually governs the contract. Every template above
   has a bracketed placeholder waiting on this.
2. **If Namibia-registered: does the Namibian ETA's cross-border consumer-protection
   reach change anything about which jurisdiction should be named as governing law?**
   Genuine legal question, not a copy one — ask counsel.
3. **Information Officer** — who is it, and what's their contact email? Required for
   the Privacy Policy and for registering with South Africa's Information Regulator
   (near-certain given the POPIA analysis above, but confirm with counsel).
4. **Payment provider** — still unresolved per [[Money & Currency Rules]] (Stitch/FNB
   Namibia/DPO Group). Terms of Service section 4 and any PCI-type claims are blocked
   on this.
5. **Shipping SLA numbers** — [[Cross-Border & Customs]] already lists this as open;
   the Shipping Policy can't state real delivery timeframes until it's answered.
6. **Restricted/prohibited goods list** — who owns this, and what's on it?
7. **Data retention schedule** — how long is account/order data actually kept? Needed
   for the Privacy Policy's retention section.
8. **Governing law / dispute resolution** (Terms of Service section 8) — cannot be
   filled in until Q1 is answered.

## Vertical slices → Trello cards

Sequenced so the legal-review-blocking items surface early and the code/copy work that
doesn't depend on them can proceed in parallel.

1. **LC-1** — Human decision gate: legal entity facts, Information Officer, governing
   law (Open Questions 1–3, 8). No code — a checklist card for Michael + counsel.
2. **LC-2** — Build `/legal/privacy` + `/legal/cookies` pages from the templates above,
   wire the consent banner's link, using placeholder brackets until LC-1 resolves them.
3. **LC-3** — Signup consent checkbox + logged acceptance (audit-log wiring).
4. **LC-4** — Build `/legal/terms` (Terms of Service) from the template, wire
   login/register's dead "Terms/Privacy" snackbar links to it.
5. **LC-5** — Build `/legal/shipping` + `/legal/returns` pages; shipping numbers stay
   placeholder pending [[Configurable Shipping Fee]] Q1/Q2 and Open Question 5 above.
6. **LC-6** — Build `/legal/customs` (Export & Customs Terms) page.
7. **LC-7** — Build the Vendor Agreement page/flow; likely surfaced during vendor
   onboarding (`/vendor/apply`), not just a footer link — decide placement at card time.
8. **LC-8** — Footer/nav wiring: replace the dead "Shipping Policy"/"Terms of Trade"
   `href="#"` links from [[Landing Site Migration]] LSM-6 with the real routes this
   spec creates.

## Implementation Notes (2026-08-26)

**Shipped:** Branch `feat/QQKjOiEH-legal-policy-pages`, bundled PR covering **LC-2, LC-3, LC-4, LC-5** (LC-1 deliberately held — legal entity facts are a human decision gate, not buildable). The five pages live under `apps/web/src/app/features/legal/` and all five routes (`/legal/privacy`, `/legal/cookies`, `/legal/terms`, `/legal/shipping`, `/legal/returns`) are `RenderMode.Prerender` matching the existing `/about`/`/services`/`/contact` pattern. A shared `.legal-*` primitive block was added to `apps/web/src/styles.scss` alongside `.marketing-*`; the five pages carry no per-page stylesheet.

**Consent capture (LC-3):** `RegisterRequest` gained a required `acceptedTerms: boolean`. `RegisterDto` validates with `@IsBoolean() + @Equals(true)`. `AuthService.register` logs a `user.terms_accepted` audit entry via `AuditService` (new `AuditAction.USER_TERMS_ACCEPTED`), writing `userId` and `entityId` both as the new user's id, `entityType: 'user'`, and metadata naming the accepted documents. Signup form's checkbox is required and not pre-ticked (`Validators.requiredTrue` — plain `required` passes on `false`).

**Dead links wired:** Consent banner's "See our POPIA notice for details" now links to the real Cookie Policy. Login's "Terms of Trade" / "Privacy Policy" stopped firing "coming soon" snackbars and became real links; register had no such links to rewire — its new consent checkbox carries them instead.

**Where code contradicted templates (templates were wrong):** The Cookie Policy template listed "remember your cart" as a cookie purpose — the cart is server-side only; removed from the published page. Only two cookies actually set: `RefreshToken` (httpOnly session) and `g_oauth_state` (OAuth CSRF, 10-minute, httpOnly, SameSite=Lax); the page names both. Google Analytics is not running (`gaMeasurementId = ''` in both environments); the page states plainly no analytics cookies are set today. `AnalyticsService` sets no cookies but does keep `hb.analytics.sessionId` in `localStorage` — a new "browser storage we use that isn't cookies" section added. `ConsentService` gates `GoogleAnalyticsService` only; the first-party `AnalyticsService` has no consent check and fires on every visit, so its lawful basis is **legitimate interest, not consent** — the Privacy Policy was reworded accordingly (template's "only if you've consented" line was false). Shipping fees resolve on `(originCountry, destinationCountry, currency)` plus per-product overrides, charging the **MAX across cart lines, never the sum** — the template's "beyond Windhoek at additional cost" implied address-based surcharge that the engine cannot charge; page was reworded and the MAX-not-sum rule published. No customer view renders `listingType` — template's Terms §2 line "your order will show which kind each item is" was false; replaced with the truth: a Marketplace product's page shows its vendor storefront. Template promised policy changes would be "flagged in-app and by email" — no such mechanism exists; reworded. Template's Privacy Policy claimed "device/browser type" is collected — `analytics_events` has no such column; removed. Contact-form submissions (`contact_inquiries`: name, email, phone, message) were a real collection category the template omitted and were added.

**Placeholders that render in the pages:** Eleven bracketed tokens in `<span class="legal-placeholder">` exist throughout: `[LEGAL ENTITY NAME]`, `[REGISTRATION NUMBER]`, `[REGISTERED ADDRESS]`, `[INFORMATION OFFICER NAME]`, `[INFORMATION OFFICER EMAIL]`, `[GOVERNING LAW JURISDICTION]`, `[RETENTION SCHEDULE]`, `[PAYMENT PROVIDER]`, `[SHIPPING FEE AMOUNT]`, `[DELIVERY TIMEFRAME]`, `[EXCLUSIONS LIST]`. Pages publish clean with no draft/pending-review banner — the exposure (unreviewed ToS reads as binding) was raised to the owner and confirmed.

**Known gaps (recorded, not hidden):** `AuditService.log` is best-effort by design and swallows its own errors so business actions never fail on a bad audit write — for a consent record that is weaker than "durable"; follow-up card exists. Google OAuth signup captures no consent: `validateOAuthLogin` creates accounts without passing through `register()`, so checkbox and audit never happen on that path; follow-up card. LC-6 (Export & Customs Terms) and LC-7 (Vendor Agreement) remain unbuilt; nothing links to `/legal/customs` or `/legal/vendor-agreement`. LC-8 (footer wiring) untouched; footer's "Shipping Policy" and "Terms of Trade" links are still `href="#"` even though both pages now exist.

**Open Questions status:** Q3 (Information Officer) and Q1 (legal entity facts) are now **load-bearing on shipped pages**, not just drafts. No questions were resolved; all remain open and blocking further work on LC-1, LC-6, LC-7.

## Implementation Notes (2026-08-27)

**Shipped:** Branch `feat/wokJ3PfW-legal-pages-and-consent-durability`, bundled PR covering **LC-6, LC-7, LC-8, LC-9, LC-10**. The two remaining pages live at `apps/web/src/app/features/legal/export-customs/` and `.../vendor-agreement/`, routed at `/legal/customs` and `/legal/vendor-agreement`, both `RenderMode.Prerender` — ten static routes now prerender. Both reuse the shared `.legal-*` primitives added in the LC-2..LC-5 batch and carry no per-page stylesheet.

**LC-6 (Export & Customs Terms):** The one thing this page could not do was flatten SACU into a blanket "no duty" claim, and it does not: SA-origin goods crossing duty-free and Procurement Service items sourced from Amazon/Shein/Temu attracting Namibian duty and VAT are stated as two distinct outcomes, with a test asserting the blanket claim never reappears. Exporter/importer of record, the prohibited-goods list, and the courier ship as four visible placeholder tokens — `[EXPORTER OF RECORD]`, `[IMPORTER OF RECORD]`, `[PROHIBITED GOODS LIST]`, `[COURIER]`.

**LC-7 (Vendor Agreement):** Placement decided at implementation time as the owner's call — a **required acceptance checkbox gating `POST /vendors`**, not a link-only page. `CreateVendorRequest` gained a required `acceptedTerms: boolean` (`@IsBoolean()` + `@Equals(true)` on the DTO); `VendorsService.create` destructures it out before building the row and writes a new `vendor.terms_accepted` audit entry. `AdminCreateVendorRequest` deliberately `Omit`s the field: an admin creating a vendor row on someone's behalf is not that vendor accepting, and recording it as if they had would be a false consent record. One new placeholder: `[TERMINATION TERMS]`.

**LC-8 (Footer wiring):** Shipping Policy, Terms of Trade and Export Documentation (relabelled "Export & Customs Terms" so the link says what it opens) now point at real routes. Returns & Refunds and the Vendor Agreement were built but unlinked and are added. Privacy and Cookie Policy are new footer additions — before this they were reachable only from the consent banner, which disappears once a visitor has chosen. "Success Stories" stays a dead `href="#"` on the LSM-6 precedent, and a test asserts it is the only one left.

**LC-10 (Durable acceptance):** New `users.termsAcceptedAt` column, written in the **same INSERT** as the account row, so an account cannot exist without its acceptance timestamp — if the write fails, registration fails. Chosen over a shared transaction (needs `QueryRunner` plumbing across `UsersService` and `AuditService`) and over a `logOrThrow` variant, which would throw *after* the user row was already committed and so leaves an orphan account unless rollback is added too. **This overrides this note's own preference for reusing the audit log rather than adding user fields** — an explicit owner decision, recorded on the card, not a drive-by. `AuditService.log` keeps its best-effort semantics untouched for every existing caller; the audit entry now records *which* documents were accepted and shares one server-derived instant with the column.

**LC-9 (OAuth consent):** `validateOAuthLogin` still creates the account but deliberately does **not** set `termsAcceptedAt` — stamping one would manufacture consent nobody gave. `AuthUser` and `UserDto` carry `termsAcceptedAt` as an ISO string or `null` (never absent; the client gate fails closed on a missing key), `authGuard` holds any signed-in account with no record at `/accept-terms` carrying the attempted URL as `returnUrl`, and `POST /auth/accept-terms` stamps the timestamp plus an audit entry marked `via: 'interstitial'`. It is idempotent — a second call keeps the original timestamp, since the first acceptance is the one with evidentiary value — and deliberately *not* best-effort: a failed write fails the call rather than telling someone their acceptance was recorded when it was not.

**Where code contradicted the templates (templates were wrong):** This note claimed the `customsReference`-before-leaving-`at_border` rule is "a real business rule already enforced in the order/shipment data model". It is **not enforced anywhere** — there is no shipping service, only `Shipment` plus a stub `SHIPPING_PROVIDER` port, and the column is nullable. Nor is customs status visible to customers: no tracking surface exists, so the page says asking support is the way to find out where a cross-border order is. Orders carry only `subtotal`, `shippingTotal` and `total` — there is no duty or tax line anywhere in pricing — so the page states that duty is not calculated, quoted, or collected at checkout instead of implying a pass-through mechanism exists. No vendor cancellation fee exists in code, so the agreement says none is charged rather than inventing an amount. The onboarding header's "Apply to become a **verified** vendor" was the same KYC overstatement the Vendor Agreement had to avoid, and was corrected.

**Existing accounts (LC-9 AC 4):** No database backfill — pre-existing accounts keep `termsAcceptedAt = null`, because fabricating a timestamp for an account that never consented is worse than an honest null. The interstitial keys on "has no acceptance record", which is exactly that state, so those accounts are **asked once on next use** rather than silently exempted. That includes the seeded bootstrap admin, which correctly has no record. Nothing is deployed, so every such account today is dev/test data; if the platform is ever deployed with real accounts predating this, the decision needs revisiting. (The first card comment said "not prompted"; review caught the contradiction with the shipped guard and the record was corrected on the card.)

**Known gaps (recorded, not hidden):** The terms gate is enforced in the **Angular router only** — a caller holding a valid access token can still reach the API without an acceptance record. Server-side enforcement was out of LC-9's scope and is commented as such in the guard. The vendor-side acceptance still rides the best-effort audit log alone; only the *signup* record got the durable column.

**Verified against a live database (2026-08-27, after the PR opened):** migration `1788172800000-UserTermsAcceptedAt` applied cleanly to the populated dev schema (5 existing accounts), reverted cleanly, and re-applied — the column lands as nullable `timestamptz` with no default and **no row was backfilled**, confirming the grandfathering decision holds in practice, including for the seeded admin. Exercised end to end against the running API: `POST /auth/register` rejects a missing or `false` `acceptedTerms` with 400 and creates no account, and on success writes the column and the audit metadata with the *same* instant; clearing the column to simulate an OAuth account makes the API report `termsAcceptedAt: null`, and `POST /auth/accept-terms` stamps it with an audit entry marked `via: 'interstitial'`, is idempotent on a second call, and 401s unauthenticated. `POST /vendors` rejects a missing or `false` `acceptedTerms` with 400, writes a `vendor.terms_accepted` audit entry on success, and the `vendors` table has no `accepted*` column — the flag never reaches the row. Verification rows were removed afterwards; the dev database is back to its five original accounts.

**Commission rate confirmed:** the live `commission_rates` table holds exactly one row — `15.00`, effective 2026-07-07, "Initial platform commission rate (provisional)" — so the **15%** published on the Vendor Agreement is the genuine in-force rate, not an assumption from the migration source. LC-7's acceptance criterion is met on evidence.

**Open Questions status:** none resolved. Q1/Q3 (entity facts, Information Officer) remain load-bearing and now block sixteen placeholder tokens across seven published pages. Q6 (restricted/prohibited goods) and the Cross-Border & Customs open questions (courier, customs documentation, exporter of record) are now visible to customers as placeholder text on `/legal/customs` rather than silently absent. LC-1 is still the gate on all of it.
