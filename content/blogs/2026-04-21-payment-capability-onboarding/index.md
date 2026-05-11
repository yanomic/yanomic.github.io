---
draft: true
title: Payment Capability - Onboarding
slug: payment-capability-onboarding
date: 2026-04-21T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Onboarding
---

## Introduction

This post addresses **capability onboarding**: deciding **what an enrolled merchant may newly accept**. The merchant is already on file with the PSP; capability onboarding only widens scope — connecting the merchant to a new acquirer, enabling a new payment method, or extending coverage to a new currency, country, scheme, or MCC. Initial PSP enrollment is a different topic: sales-led, contract-driven, and largely off-API.

### Counterparties and Rules

The counterparties are the merchant and an **acquirer** (or wallet operator, bank, LPM partner), usually fronted by a **gateway / PSP** integration layer. **The shopper is not involved** at onboarding time — the application sets the boundaries the merchant can later initiate against at checkout.

On the card side, the ground rules are shared: **scheme acceptance rules** and the merchant's **PCI program** are the same reference set across brands. **LPMs** have no equivalent: each bank, wallet operator, or platform sets its own requirements, and the same rail can vary by country or tier.

### What the Merchant Submits

The information the merchant must supply ranges from **structured legal documents** (incorporation certificate, ownership, directors' IDs, licenses, bank statements, processing history) to **free-text business descriptions** (what you sell, how you fulfill, refund policy, risk controls). For a merchant already on file, the request is the **delta** — KYC and entity material the PSP already holds is reused; only what the new capability needs is collected fresh.

Automation is uneven. Card acquirers typically expose an **onboarding API** or at least a structured partner portal for submission. For many LPMs, the practical channel is still **email threads** or **shared Google Docs / spreadsheets**, with a human on the other end requesting clarifications. Even where an API exists, the review itself is not synchronous — the acquirer or bank takes time to underwrite, and outcomes span **approve**, **approve with conditions / partial approval**, **need additional documents**, and **reject**. Merchants must not assume the API response is the final word, and must instead rely on a status channel: a query endpoint, a webhook, or both.

### What Approval Produces

A granted capability resolves into one of two identifier outcomes:

- **New counterparty connection**: a fresh identifier is provisioned by the party being connected. For an acquirer, this is a **MID** (Merchant ID), often arriving as a **hierarchy** — one or more MIDs (by scheme, currency, or legal entity) with **store IDs** and **terminal IDs (TID)** beneath them. For an LPM partner, the identifier follows that partner's own naming — **partner ID**, **merchant code**, **app ID**, **handle**, **VPA** — and there is no universal MID equivalent.
- **Scope extension on an existing connection**: no new identifier; the existing MID or partner-side identifier's allowed scope is widened (additional currency, additional brand, raised volume cap). The merchant sees a status change in the PSP's records, not a new identifier.

When a PSP aggregates acquirers on your behalf, the **PSP-level merchant identifier** is the primary handle you keep; the underlying acquirer MIDs are managed by the PSP and may not be directly visible.

End-to-end, onboarding can take anywhere from **days** (well-prepared low-risk merchant on an integrated PSP) to **several months** (regulated category, multi-region, multiple LPMs in parallel). The dominant cost is human review over non-standard inputs.

## Integration Patterns

Onboarding is delivered through three patterns that share a single API surface underneath:

- **Embedded**: the provider renders UI inside the merchant's surface, drop-in components, web SDKs, or iFrames (Stripe Connect Embedded Components, Adyen Onboarding component, Airwallex Embedded Onboarding). The merchant keeps control of the page and navigation; the provider owns form rendering, document capture, and consent screens.
- **Hosted**: the merchant redirects the user to a provider-hosted onboarding page (Stripe hosted onboarding links, Adyen Hosted Onboarding Page, Airwallex Hosted Onboarding). The provider owns the entire flow until the user returns.
- **API**: the merchant collects every field and document in its own UI and submits via REST. The merchant owns rendering, validation, document capture, and consent capture.

> Embedded and hosted are presentation layers over the API. **Feature parity** across the three comes from routing all of them through the same requirements, application create and patch, submit, application reads, and documents operations under the hood. Those two patterns are initiated through a **short-lived session**: the merchant obtains a signed URL or client secret so the provider UI can run. Most teams pick embedded or hosted to avoid owning three things they rarely want: dynamic field rendering, document capture and OCR, and consent flow rendering (terms, data-sharing, scheme disclosures).

### Application Target

Each onboarding **application** is filed against a **`target`**: the noun being provisioned. Model `target` as a top-level field. Two parts are required; `scope` is optional and varies by `target.type`.

- **`target.type`** (required): typed discriminator. Candidate values: `paymentMethod`, `region`, or more.
- **`target.code`** (required): identifier within that type. Sample values: `klarna`, `us`, `sg`.
- **`target.scope`** (optional): refines the ask. Omitting `scope` entirely requests the broadest provisioning the target supports.
    - When `target.type` is **`paymentMethod`**:
        - **`scope.currencies[]`** (optional): ISO 4217 list. Omit → all currencies the PSP offers for that method.
        - **`scope.provider`** (optional): routes to a named acquirer or partner. Omit → PSP default.
    - When `target.type` is **`region`**:
        - **`scope.paymentMethods[]`** (optional): each item is an object. Omit → every method available in that region.
            - **`code`** (required per item): payment method code within the region.
            - **`currencies[]`** (optional per item).
            - **`provider`** (optional per item).

> Production `scope` payloads carry many more fields than this sketch. Prefer **nested objects and object arrays** over flat scalars throughout `scope`, so providers can add or retire qualifiers without breaking the outer shape.

A single **`target`** can fan out to several internal capabilities: a `region` expands into multiple acquirer or method capability rows under that geography; an umbrella payment-method group expands into per-partner registrations the PSP files separately. When internal legs succeed or fail independently, the merchant sees a mixed grant — some legs live, others still blocked.

### Functional Endpoint Inventory

The endpoints below drive an application through its lifecycle and read state off the resulting merchant enrollment. The state model they operate against:

#### Onboarding State Machine

The same state vocabulary applies at two levels: an aggregate `state` on the application resource, and — when a `target` fans out — a per-leg `state` on each entry in the `details` array for an internal capability. Providers may roll legs up into the aggregate or expose both; integrations should read both when present. The states worth modeling explicitly are:

- `Created`: the application resource exists with a **fixed** **`target`**; the merchant may patch `data` and attach documents before filing for review. Nothing is queued to underwriting yet; per-leg `details` are usually absent or empty until the application reaches `Submitted`.
- `Submitted`: application has been filed; not yet picked up for review.
- `UnderReview`: the acquirer / PSP underwriting team is actively assessing the application.
- `InfoRequested`: the reviewer has asked for additional documents or clarifications; the ball is in the merchant's court. Returns to `UnderReview` once the merchant responds.
- `Approved`: underwriting passed for every internal capability the requested scope expanded into (full requested scope: brands, MCCs, currencies, geographies, partner registrations).
- `PartiallyApproved`: underwriting passed for a strict subset of the requested target's expansion — for example, a `region` target where some method/acquirer pairs cleared and others did not, or an umbrella payment-method group where some partner registrations succeeded. The application is not rejected, but later transactions outside the approved subset will be declined.
- `Rejected`: underwriting refused; no scope is granted. Usually terminal for this application; some providers support appeal or reopen paths, while others require a new application.


```mermaid
stateDiagram-v2
    [*] --> Created: application created
    Created --> Created: PATCH save
    Created --> Submitted: submit for review
    Created --> [*]: discard
    Submitted --> UnderReview: review starts
    UnderReview --> InfoRequested: reviewer needs more
    InfoRequested --> UnderReview: merchant responds
    UnderReview --> Approved
    UnderReview --> PartiallyApproved
    UnderReview --> Rejected
    Approved --> [*]
    PartiallyApproved --> [*]
    Rejected --> [*]
```

Until aggregate `state` reaches `Submitted`, internal fan-out rows are not in play; checkout and listing still read the merchant's **already granted** scope from a provisioning read or projected catalog, not from the unsent application.

#### Endpoints

##### Requirements

- `POST /onboard/requirements/lookup`: returns the requirements for filing an application against a given **`target`**.
    - **Request**: the **`target`** object. Every qualifier the application needs is nested inside `target`.
    - **Response**: an ordered list of requirement entries describing exactly what the caller must supply.
        - **`key`**: dotted path under which the value is submitted.
        - **`name`**: short human-readable label, suitable for a form control.
        - **`description`**: longer help text — what the reviewer expects, examples, gotchas.
        - **`requirement`**: one of `required`, `optional`, or `conditional`. `conditional` carries a **`condition`** sub-object.
        - **`type`**: the value kind — `string`, `number`, `boolean`, `enum`, `document`, `consent`, `object`, or `array`. 
        - **`format`** (optional): per-type constraints layered on `type` — for example `enum.values`, `string.pattern` / `minLength` / `maxLength`.

##### Application

- `POST /onboard/applications`: creates an onboarding application **without** submitting it for review. The merchant receives an `applicationId` and aggregate `state` `Created`; underwriting does not start until **`POST /onboard/applications/{applicationId}/submit`**. The **`target`** set here **does not change** on later PATCH or submit calls — if the merchant needs a different provisioning unit, they discard this application and create another. 
    - **Request**: **`target`** (required): the **`target`** object for this application — the same shape used with `POST /onboard/requirements/lookup`. **`data`** (optional): requirement values keyed by paths from that lookup for this target; omit fields the UI has not collected yet.
    - **Response**: the application resource — `applicationId`, `state`: `Created`, and usually no `details` fan-out until submit.
    - **Errors**: a malformed **`target`** returns `422 Unprocessable Entity` with structured paths; no `applicationId` is minted until create succeeds.
- `POST /onboard/applications/{applicationId}/submit`: validates the record and transitions `Created` → `Submitted` on the same `applicationId`, enqueueing underwriting. The stored **`target`** is authoritative; the submit body does not replace it.
    - **Request**: a `data` map carrying values for every non-document requirement entry returned by `POST /onboard/requirements/lookup` for the application's **`target`**. Document requirements are satisfied separately via `POST /onboard/documents` before submit when they are `required` — the request body carries the `applicationId` and the requirement `key`; document resources are not nested under the application path.
    - **Response**: the application resource — `applicationId`, current overall `state`, and an optional `details` array carrying the per-capability fan-out. Each `details` entry covers one internal capability the target expanded into and carries a stable capability identifier and that leg's own `state` using the same vocabulary as the aggregate.
    - **Errors**: submitting with any `required` non-document requirement missing — or any `conditional` requirement whose `condition` evaluates `true` but whose value is missing — returns `422 Unprocessable Entity` with the violating requirement entries in the body. Aggregate `state` remains `Created` until submit succeeds. Calling submit when aggregate `state` is not `Created` returns a client error (for example `409 Conflict`) with a structured reason.
- `GET /onboard/applications/{applicationId}`: reads the application resource.
    - **Response**: the application resource — `applicationId`, current overall `state`, and the optional `details` array of per-capability fan-out. Per-entry `state` and `reason` reflect each leg's latest outcome. While any leg is still pending, the aggregate `state` should remain `UnderReview`; once every leg has reported, the aggregate shows `Approved`, `PartiallyApproved`, or `Rejected`, with structured rejection reasons where applicable.
- `GET /onboard/applications/{applicationId}/requirements`: returns **additional** requirements raised by the LPM or acquirer during review — data, documents, or consents that were not part of the original `POST /onboard/requirements/lookup` response and that the underwriter now needs before deciding. This is the read the merchant relies on when the application sits in `InfoRequested`. Supplying the requested values via `PATCH /onboard/applications/{applicationId}` (data) or `POST /onboard/documents` (documents) drives the application back to `UnderReview`.
    - **Response**: same entry shape as `POST /onboard/requirements/lookup`, so the same renderer handles both. Providers may also surface this list as a sub-field on the application read.
- `DELETE /onboard/applications/{applicationId}`: withdraws the application before decision, including deleting an unsubmitted application in `Created`.
- **Webhooks**: the provider's push channel so the merchant learns when asynchronous review moves an application — an outbound `POST` to the merchant's registered URL.
    - **Payload**: JSON body in the same shape as the `GET /onboard/applications/{applicationId}` response, reflecting the application after the `state` change that triggered the webhook (for example into `Submitted` from `Created`, into `InfoRequested`, or to `Approved` / `PartiallyApproved` / `Rejected`). Many providers omit push for intermediate `Created` PATCH saves and only notify once the filing enters `Submitted` or later.
    - **Acknowledgement**: the merchant responds with `200 OK` with a fixed-contract body the provider documents; other status codes or malformed acks typically trigger retries.
- `PATCH /onboard/applications/{applicationId}`: idempotent update of `data` values. **`target`** is not patchable after create. While aggregate `state` is `Created`, PATCH is the save path before **`POST /onboard/applications/{applicationId}/submit`**. After submit, patching a specific artifact is the supported retry pattern during `InfoRequested`; do not open a second application for the same ask. When the application is in `InfoRequested`, a successful patch automatically transitions it back to `UnderReview` — there is no separate submit step. Document requirements are not patched here; use the Documents endpoints.
    - **Request**: a body keyed by requirement `key` paths (e.g. `{ "legalEntity.registrationNumber": "..." }`).

##### Documents

- `POST /onboard/documents`: uploads a document and binds it to a specific document requirement on an application. The path does not embed `applicationId`; the caller supplies it alongside the requirement `key` in the multipart or companion metadata block. The same call applies while aggregate `state` is `Created`, once the merchant holds an `applicationId`.
    - **Request**: multipart upload — `applicationId`, `key` (the document requirement being satisfied), the file binary, and optional metadata (`type`, `purpose`).
    - **Response**: a `documentId` for the stored document. The corresponding requirement entry flips from unsatisfied to satisfied; if this completes the application's outstanding requirements while it sits in `InfoRequested`, the application transitions back to `UnderReview`.
- `GET /onboard/documents/{documentId}`: reads document metadata and review state.
- `DELETE /onboard/documents/{documentId}`: removes a document. Meaningful only before the underwriter commits to a decision.

## The Five Lenses

- **Semantics**: answer one question: *"May this merchant submit transactions of type X, in country Y, on rail Z?"* Output is a scope grant on the existing enrollment — additional brands, MCCs, currencies, geographies, or partner registrations — plus, when the requested target fans out to a new acquirer connection, a fresh credentialed identity (MID or LPM-equivalent).
- **State model**: the state machine above is the source of truth from `Created` through terminal underwriting outcomes. `Created` is pre-review: the merchant edits freely until a successful `POST /onboard/applications/{applicationId}/submit`; only `InfoRequested` is non-final once review has started — underwriting pauses until the merchant supplies what was asked, then the application returns to `UnderReview`. When the `target` fans out, the aggregate stays `UnderReview` until every leg has reported, then rolls up to `Approved`, `PartiallyApproved`, or `Rejected`.
- **Recovery**: the merchant-side retry loop is **resubmitting a specific artifact**, not re-filing a new application for the same ask. If the merchant creates another application whose `target` is the same as, overlaps, or lies inside an application already in flight for that merchant — including one still in `Created` — the provider should reject the new application and return a pointer to the existing `applicationId` (typically `422` with a structured reason), not mint a second record. Well-designed onboarding APIs anchor on an **application id** (idempotent updates), expose **per-artifact upload endpoints**, and return **structured rejection reasons** ("license document unreadable", "MCC not permitted") so follow-up can be automated instead of email-threaded.
- **Time discipline**: review SLAs are bounded for cards at PSPs that publish one (hours to a few days) and mostly unbounded for manually handled LPMs. **Document validity** windows apply, so long-paused applications force re-collection. Some programs require periodic re-validation, including annual PCI evidence cycles and, in specific cases, scheme-program renewals. When the grant becomes effective for live traffic is rarely a first-class field on the application resource; it follows review outcome plus scheme, acquirer, or PSP operational rules (activation, clearing cutoffs, MID boarding lag).
- **Observability**: two modes, both required: a **status query** on the application or a **provisioning read** of the merchant as the source of truth, and **status webhooks** for transitions (`Submitted`, `InfoRequested`, `Approved`, `PartiallyApproved`, `Rejected`). Long-term observability also needs a **provisioning read API** — which capabilities are active for which merchant, in what scope, with which underlying MIDs or partner identifiers — because staleness here causes unexplained declines downstream.

## Summary

Capability onboarding extends an enrolled merchant so it can initiate or accept payments on additional rails and payment methods, in new geographies and currencies, after the provider approves the widened scope. The same underwriting and filing work runs whether the merchant uses embedded components, a hosted redirect flow, or a native API — only the presentation layer changes.

That provisioning is the foundation for payment checkout: listing and selecting payment methods at runtime draw on a catalog that *projects* the granted capability set, filtered for the session (amount, currency, country, channel, shopper).


## Appendix

### Appendix A: Provider documentation

- **Stripe (Connect):** [Connect capabilities](https://docs.stripe.com/connect/account-capabilities), [Update a capability](https://docs.stripe.com/api/capabilities/update), [Update merchant profile](https://docs.stripe.com/api/accounts/update), [Handle verification with the API](https://docs.stripe.com/connect/handling-api-verification).
- **Adyen (Platforms / LEM):** [Legal Entity Management v4](https://docs.adyen.com/api-explorer/legalentity/latest/overview), [Update a legal entity](https://docs.adyen.com/api-explorer/legalentity/latest/patch/legalEntities/_id_), [Capabilities for Adyen for Platforms](https://docs.adyen.com/marketplaces-and-platforms/verification-overview/).
- **Airwallex (connected merchants):** [Capabilities](https://www.airwallex.com/docs/connected-accounts/about/account-capabilities), [Payment methods list](https://www.airwallex.com/docs/connected-accounts/onboarding/kyb-and-onboarding/native-api/payment-method-list), [Native API onboarding](https://www.airwallex.com/docs/connected-accounts/onboarding/kyc-and-onboarding/native-api).
- **Checkout.com (Platforms):** [Onboard sub-entities](https://www.checkout.com/docs/platforms/onboard-sub-entities), [Submit application via API](https://www.checkout.com/docs/platforms/onboard-sub-entities/submit-application/onboard-with-the-api), [Handle webhooks](https://www.checkout.com/docs/platforms/onboard-sub-entities/handle-webhooks), [Add supporting documents](https://www.checkout.com/docs/platforms/for-payfac/onboard-sub-entities/add-supporting-documents).
- **Worldpay (Access, parties model):** [Parties API](https://docs.worldpay.com/access/products/parties), [Access APIs index](https://docs.worldpay.com/access/apis), [Worldpay for Platforms onboarding](https://resource-center.worldpayforplatforms.com/wp4p/merchant-onboarding-via-api).
- **Worldline (Onboarding API, NAM docs):** [Onboarding API introduction (NAM)](https://docs.na.worldline-solutions.com/build-your-integration/onboarding-api/onboarding-api-introduction/), [Onboarding API overview (NAM)](https://docs.na.worldline-solutions.com/build-your-integration/onboarding-api/onboarding-api-introduction/overview), [Worldline Connect API Explorer](https://docs.connect.worldline-solutions.com/documentation/api-explorer).
- **Antom (Merchant Service / AMS):** [Merchant onboarding](https://docs.antom.com/ac/merchant_service/merchant_onboard), [Merchant administration](https://docs.antom.com/ac/merchant_service/account_manage), [Merchant registration API](https://docs.antom.com/ac/ams/registration).

### Appendix B: Sub-merchant onboarding on Platforms

**Sub-merchant onboarding** is how a platform brings sellers or service providers under its payments program when those parties have no direct acquirer or LPM contract. The platform (or a PSP acting as its infrastructure partner) is the counterparty to the scheme and the bank; it runs or delegates underwriting, owns most of the KYC/KYB lifecycle, and carries scheme-level registration and monitoring obligations for the population beneath it. The same basic fact — the seller is not the contracted merchant of the acquirer — shows up in four common platform models:

- **Payment Facilitator (PayFac)**: the platform holds a scheme PayFac license and operates a master MID under a sponsoring acquirer; sub-merchants are Visa "Sponsored Merchants" beneath it, with the PayFac as the contracted merchant of the acquirer. The PayFac runs its own underwriting and assigns sub-MIDs internally. Examples: Toast, Square.
- **Connect-style PSP service**: a PSP exposes PayFac infrastructure that other platforms build on. The PSP is the licensed PayFac; the platform integrates rather than holds the license itself, and the seller onboards into the PSP's connected-merchant model through the platform's UI or API. Examples: Stripe Connect, Adyen for Platforms, Airwallex connected merchants, Checkout.com sub-entities. Consumer platforms layered on top (for example Shopify Payments on Stripe Connect) inherit this shape.
- **Marketplace** (a defined Visa entity type): platforms that sell on behalf of sellers under Visa's Marketplace classification. Visa applies a separate marketplace responsibility framework with its own retailer/marketplace registration and dispute handling, but the seller still has no acquirer relationship. Examples: Uber, Airbnb, Etsy.
- **Merchant of Record (MoR)**: the platform *is* the merchant for every transaction; vendors underneath have no scheme relationship at all and onboard for tax remittance, payout, and AML — not card acquiring. The MoR's name appears on the cardholder statement; chargebacks, refunds, and tax obligations land on the MoR, not the vendor. Examples: Paddle, Lemon Squeezy, FastSpring.

Each model carries its own KYC/KYB obligations and lifecycle, and inherits **scheme-level sub-merchant registration** rules: above scheme-defined thresholds (historically a $1M annual sub-merchant volume baseline, with Visa's October 2024 update adding MCC-list and established-relationship carve-outs), the platform must register the sub-merchant directly with the scheme rather than carry it as an aggregated party. Current thresholds and exceptions should be checked against the latest scheme bulletins.

The exception is the **ISO / referral model**, where each sub-merchant signs its own acquirer contract and the platform is a sales or integration channel only. The seller has a direct acquirer relationship in this case, so the lifecycle collapses back into the single-merchant capability onboarding the sections above describe.


