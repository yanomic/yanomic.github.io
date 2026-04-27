---
draft: true
title: Payment Capabilities
slug: payment-capabilities
date: 2026-04-20T20:32:28+0800
categories:
- Payments
tags:
- Payment Capabilities
- Payment Lifecycle
- Payment Ecosystem
---

## Introduction

Payment methods do not behave the same way. The split between cards and local payment methods is the obvious one, but each category hides its own variations.

This creates real friction on both sides. Merchants want to accept payments reliably. Shoppers want to pay with their preferred method. Both sides end up navigating inconsistent capabilities, edge cases, and operational trade-offs they should not need to know about to complete a simple transaction.

This blog series focuses on merchant-facing APIs and describes capabilities from a functional perspective — the observable inputs, outputs, and behavior of each payment "black box." Provider-specific implementation details are out of scope.

We also compare payment methods and PSPs against a standard set of capabilities. The goal is a common integration model that absorbs method-specific characteristics rather than leaking them into merchant code.

## Ecosystem and Lifecycle

Card payments are a useful reference because the roles are well defined and the same lifecycle vocabulary (authorize, capture, refund, dispute) appears across many integrations. The ecosystem is not just "shopper and merchant"; several specialized parties cooperate, each with a narrow responsibility.

**Typical parties in a card transaction**

- **Cardholder**: The person whose card is used; proves identity and consent (3-D Secure, CVV, PIN where applicable) and holds the account at the **issuer**.
- **Merchant**: Sells goods or services and initiates payment requests toward its **acquirer** (directly or through a **gateway / PSP**).
- **Acquirer** (merchant bank): Underwrites the merchant, routes authorization and capture messages, receives settlement from the card network, and credits the merchant (minus fees). 
- **Scheme** (Card network): Visa, Mastercard, and others. Sets rules, routes messages between acquirer and issuer, and operates clearing and often parts of dispute handling. Networks do not hold the cardholder's money; they coordinate messaging and settlement between banks.
- **Issuer** (cardholder bank): Issues the card, approves or declines authorization based on risk and available funds, posts charges to the cardholder, and participates in clearing, settlement, and disputes on the cardholder side.
- **Gateway / PSP** (optional but common): Aggregates many merchants, offers a single API, tokenization, fraud tools, and connectivity to one or more acquirers. From the merchant's perspective, the gateway is often the primary integration surface even though settlement still runs through acquirer–network–issuer rails. In modern integrations the gateway and PSP are often the same product (e.g. Stripe, Adyen, Checkout.com), so for brevity we treat them as one role throughout.

With the parties in place, the same transaction flows through a sequence of well-defined phases. The sections below walk through each one in order.

### Onboarding
The merchant (optionally via PSP) onboards with an acquirer: KYC, pricing, MCC, and technical connectivity. The acquirer underwrites the merchant against **scheme rules** — PCI, branding, permitted category use, and cardholder-data handling. In the usual path the **acquirer** validates and approves the application (a **PSP** may handle operations in front of the acquirer). Direct scheme approval is the exception: some programs, high-risk categories, or network registration require the acquirer to file with the scheme, which then accepts or declines. **The cardholder is not involved**; onboarding only establishes what the merchant may initiate later at checkout.

```mermaid
sequenceDiagram
    participant Mer as Merchant / PSP
    participant Acq as Acquirer
    participant Sch as Scheme
    Note over Sch,Acq: Scheme sets rules, and the acquirer must enforce them on sponsored merchants.
    Mer->>Acq: Apply (KYC, business model, MCC, channels, volumes)
    Acq->>Acq: Underwriting, PCI / data review vs scheme requirements
    opt Registration or scheme review required
        Acq->>Sch: Merchant or program registration
        Sch->>Acq: Approve / decline
    end
    Acq->>Mer: MID, pricing, technical connectivity, cleared to accept the brand
```

### Authorization
Payment initiation is the merchant starting an authorization: assemble amount, currency, merchant identifiers, and card details, and send the **authorization request** through the gateway and acquirer toward the issuer. During that authorization, the issuer may need to **authenticate** the cardholder first (strong customer authentication / SCA — e.g. 3-D Secure, bank app, OTP) before approving. The outcome is typically structured data (e.g. CAVV, ECI) the issuer uses with funds and risk checks to **approve or decline** the transaction. If approved, the issuer places an **authorization hold**. A hold confirms availability and (when SCA ran) payer intent; it does not pay the merchant. Funding follows **capture** and **settlement**.

```mermaid
sequenceDiagram
    participant Mer as Merchant
    participant DS as 3DS Server / Directory
    participant ACS as Issuer ACS
    participant Ch as Cardholder
    participant G as Gateway / PSP
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    opt Strong customer authentication (e.g. 3-D Secure 2)
        Mer->>DS: Authentication request (device data, amount, PAN)
        DS->>ACS: Route to issuer ACS
        alt Challenge required
            ACS->>Ch: Step-up challenge
            Ch->>ACS: Authenticate
        end
        ACS->>Mer: Authentication result (CAVV / ECI)
    end
    Mer->>G: Authorization request (amount, merchant id, tokenized PAN, CAVV / ECI)
    G->>Acq: Forward auth
    Acq->>Sch: Auth request
    Sch->>Iss: Auth request
    Iss->>Iss: Decide using funds, risk, and SCA evidence
    Iss->>Sch: Approve or decline, hold if approved
    Sch->>Acq: Result
    Acq->>G: Result
    G->>Mer: Authorized or declined
```

### Capture
The merchant (or automated rules) sends **capture** instructions for all or part of the authorized amount. The acquirer **presents** those transactions into clearing: the scheme exchanges clearing records with the issuer so the charge can be posted to the cardholder. Capture is about what is owed and **moving the transaction into clearing**; it is still distinct from **settlement**, where money actually moves between banks.

```mermaid
sequenceDiagram
    participant Mer as Merchant
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    Mer->>Acq: Capture instruction (full or partial)
    Acq->>Sch: Clearing / presentment
    Sch->>Iss: Clearing records
    Iss->>Iss: Post charge to cardholder
```

### Settlement
After clearing, **interbank settlement** nets obligations between issuer and acquirer according to the scheme's arrangements. Separately, the **acquirer settles to the merchant**: payout timing, reserves, and fees are defined in the merchant's contract. The scheme orchestrates settlement between issuer and acquirer; it does not replace the acquirer's merchant payout.

```mermaid
sequenceDiagram
    participant Iss as Issuer
    participant Sch as Scheme
    participant Acq as Acquirer
    participant Mer as Merchant
    Note over Iss,Sch: Net settlement of cleared obligations
    Iss->>Sch: Member settlement leg
    Sch->>Acq: Member settlement leg
    Acq->>Mer: Merchant payout per contract
```

### Cancel / Void
If the merchant will not capture — order canceled, inventory unavailable, or duplicate auth — they **void** or **cancel** the authorization while it is still valid. The acquirer asks the issuer to **release the hold**; no capture means no clearing/settlement for that authorization. Naming varies by provider (`void`, `cancel`, `reverse authorization`); the idea is the same: end the hold without taking money.

```mermaid
sequenceDiagram
    participant Mer as Merchant
    participant G as Gateway / PSP
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    participant Ch as Cardholder
    Mer->>G: Void / cancel authorization
    G->>Acq: Forward reversal
    Acq->>Sch: Authorization reversal
    Sch->>Iss: Release hold
    Iss->>Ch: Hold removed from available balance
    Iss->>Sch: OK
    Sch->>Acq: Reversal confirmed
    Acq->>G: Canceled / voided
    G->>Mer: Final status
```

### Refund
Refunds return money to the cardholder after a successful capture. They are initiated on the **merchant/acquirer** side and ride the card rails as a **credit**; timing, partial refunds, and cutoffs depend on network and issuer rules.

```mermaid
sequenceDiagram
    participant Mer as Merchant
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    participant Ch as Cardholder
    Mer->>Acq: Refund request
    Acq->>Sch: Credit / refund message
    Sch->>Iss: Post credit
    Iss->>Ch: Cardholder sees credit
```

### Dispute / Chargeback
Unlike refunds, **disputes** start when the **cardholder** challenges the charge with the **issuer**. The issuer opens a **chargeback** (or similar) case, and the acquirer and merchant exchange evidence under **scheme** rules and timelines. Outcomes can reverse or adjust what was settled — so "payment succeeded" in an API is not always the end of **operational risk**.

```mermaid
sequenceDiagram
    participant Ch as Cardholder
    participant Iss as Issuer
    participant Sch as Scheme
    participant Acq as Acquirer
    participant Mer as Merchant
    Ch->>Iss: Dispute transaction
    Iss->>Sch: Chargeback / dispute case
    Sch->>Acq: Notify acquirer
    Acq->>Mer: Evidence request / debit notice
    Mer->>Acq: Evidence or accept outcome
    Acq->>Sch: Represent / outcome
    Sch->>Iss: Final disposition
```

### Tokenization
The lifecycle above focuses on a **one-off** payment path. This section extends that model to **stored-instrument** journeys, where a payment credential is saved first and then referenced in later transactions. This extension is easiest to read as **two connected phases**: 

1. **Token creation** — in an in-session checkout, the PSP provisions a reusable token bound to the underlying payment credential.
2. **Subsequent charge** — later authorizations reference that saved token, often without the cardholder present.


#### Phase 1: Token Creation

Phase 1 is a **customer-initiated transaction (CIT)**. Provisioning and consent are handled within the same checkout flow: the cardholder supplies card details, and the **gateway / PSP** calls a **token vault** or **network tokenization** service to create a token bound to that credential. In the same CIT flow, the shopper accepts terms to save the card, completes **SCA** when required, and the authorization carries the needed **stored credential** indicators.

```mermaid
sequenceDiagram
    participant Ch as Cardholder
    participant Mer as Merchant
    participant G as Gateway / PSP vault
    participant T as Token / network token service
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    Ch->>Mer: Pay + agree to save card / subscription terms
    Mer->>G: Authorization with stored-credential indicators (initial CIT, setup)
    G->>T: Provision token
    T->>G: Token id (PAN not stored at merchant)
    G->>Acq: Auth request with token + CIT + stored-credential (initial/setup) indicators
    Acq->>Sch: Forward
    Sch->>Iss: Authorization request
    alt SCA required
        Iss->>Ch: Step-up
        Ch->>Iss: Authenticate
    end
    Iss->>Iss: Approve / decline (funds, risk, SCA)
    Iss->>Sch: Result
    Sch->>Acq: Result
    Acq->>G: Result
    G->>Mer: Approved, token bound, stored-credential agreement for declared MIT use cases
```

#### Phase 2: Subsequent Charge

Phase 2 means **reusing the saved token** for a later charge, without collecting raw payment details again. The same tokenized pattern can be either **MIT** or **CIT**, depending on who initiates the transaction. If your backend triggers the charge while the cardholder is **not in session** (for example, subscription renewal or **unscheduled** top-up), it is a **merchant-initiated transaction (MIT)**. If the cardholder is present and explicitly confirms “pay with saved card,” it is a **customer-initiated transaction (CIT)** using the same saved token. In both cases, acquirer/scheme routing resolves the token to the underlying card for issuer decision.

```mermaid
sequenceDiagram
    participant Mer as Merchant
    participant G as Gateway / PSP vault
    participant Acq as Acquirer
    participant Sch as Scheme
    participant Iss as Issuer
    Mer->>G: Charge saved token (MIT category: recurring, UCOF, etc.)
    G->>Acq: Authorization / capture with token + MIT indicators + link to original agreement
    Acq->>Sch: Forward
    Sch->>Iss: Approve / decline (issuer MIT / risk rules)
    Iss->>Sch: Result
    Sch->>Acq: Result
    Acq->>G: Result
    G->>Mer: Final status
```

#### MIT Classification

These **business patterns** describe how **Phase 2** legs are labeled for schemes; they mostly apply when the merchant **initiates** the charge:

| Term | Meaning |
|------|--------|
| **Card on file (CoF)** | Umbrella: the card is **stored** (as a token) for later use. Does not by itself mean subscription. |
| **Subscription** | **Scheduled** charges (e.g. monthly fee): MIT with a **recurring** indicator; amount may be fixed or variable per your agreement. |
| **Unscheduled card on file (UCOF)** | Industry label for **merchant-initiated** charges **without** a fixed schedule — e.g. metered use, top-ups, “charge when balance low.” Mastercard uses “UCOF”; Visa’s aligned category is “Unscheduled Credential on File” (same acronym). Still requires valid prior **CIT** agreement from Phase 1. |

## Local Payment Methods

For local payment methods (LPMs), the **ecosystem and lifecycle are largely the same** as cards: similar participants and the same high-level phases — initiate, confirm, collect funds, reconcile, and handle refunds and problems. The **differences are in the details** — who plays each role, how authorization is triggered, and how refunds and disputes work. 

### Ecosystem and Lifecycle at a Glance

At a high level, LPMs replace cards' **single global scheme layer** with **national or regional operators** — e.g. PIX (BCB, Brazil), UPI (NPCI, India), SEPA CT / SDD (EPC), iDEAL (Currence / EPI), BLIK (Poland), Swish, Bizum, FPS, PayNow, PromptPay — each with its own rules, settlement, and dispute framework. Some LPMs are genuinely **bank-led**, **wallet- or platform-led**, or **bilateral**, with no scheme analog at all. As a result, you cannot assume one global rulebook the way you can with Visa or Mastercard, and the issuer/acquirer duality often collapses into a single bank role.

The lifecycle shape shifts accordingly: **authorization is frequently push-based** (shopper-pushed from their bank or app) rather than pull-based, and **refund/dispute frameworks are usually weaker** than the card chargeback system. Stored-instrument flows also look different — there is no global "PAN token" equivalent, so **Phase 2** (reusing a saved instrument) only exists where the rail supports **standing mandates** or **billing agreements** for repeat or merchant-initiated collection, and behavior stays **method- and country-specific** rather than a single stored-credential framework.

### Summary of Differences

The table below consolidates the card baseline and typical LPM behavior across the aspects touched on above:

| Aspect | Card ecosystem | Typical local / APM behavior |
|--------|----------------|------------------------------|
| **Who "owns" the user account** | Issuer holds the card account; network rules standardize messaging. | Often a bank, wallet, or local operator; rarely a separate “scheme” you integrate against directly. |
| **Authorization path** | Real-time approve/decline on a shared rail (network + issuer). | May be synchronous, or **pending** until the shopper completes a bank login, transfer, QR scan, or store payment. |
| **Credential** | PAN + expiry (often tokenized); strong customer authentication when required. | Bank redirect, IBAN, mobile number, voucher code, QR — method-specific. |
| **Settlement and clearing** | Highly standardized batch clearing between banks via the network. | May be instant push, batched bank transfer, or cash-agent settlement; reconciliation fields differ. |
| **Refunds and disputes** | Chargeback framework is mature and standardized; evidence windows are strict. | Refund support ranges from full to limited or manual; "dispute" may be a support ticket rather than a scheme chargeback. |
| **Merchant integration** | One mental model (auth/capture/refund) maps across many regions if the PSP abstracts schemes. | More one-off behaviors: expiry of payment codes, offline confirmation, different webhook semantics. |
| **Saved “token” / instrument id** | **Network** or **PSP token** mapping to PAN. | **Vault payment-method id**, mandate reference, wallet handle — **not** a card PAN token. |

Cards are not universally "better"; they are a **shared rail with predictable roles**. LPMs trade that uniformity for **local reach, lower cost in some markets, or shopper preference** — at the cost of more asynchronous states and method-specific operational playbooks. The next section introduces payment capabilities as the common lens for comparing methods despite these differences.

## Payment Capabilities

The **Ecosystem and Lifecycle** section above describes *who* is involved and *what happens* from onboarding through disputes. **Payment capabilities** are the complementary lens: the operations and signals merchant-facing APIs expose so each phase can be driven and observed reliably, without guesswork or manual follow-up.

Reliability here means predictable **semantics** (what a call does and does not do), a usable **state model** (statuses and allowed transitions), **recovery** (idempotency, safe retries, clear errors), **time discipline** (expiry, capture windows, settlement latency), and **observability** (queries or webhooks that reflect reality soon enough for operations). Comparing methods is not just whether capabilities exist, but how consistently they behave across error shapes, transitions, and asynchronous constraints. This is especially where LPMs stress integrations (pending flows, shopper action outside the browser, weaker refund/cancel paths), and why a **standard capability model** helps compare rails on equal footing while preserving method-specific behavior.

Typical capabilities and the reliability problem each one addresses:

- **Authorize**: Reserve funds or confirm intent before final collection, so checkout can commit without immediately moving money. Reliability requires clear outcomes and holds that match issuer behavior.
- **Capture**: Collect funds (full or partial, where supported) after authorization or confirmation, aligned with fulfillment. Reliability requires amount rules, timing discipline, and idempotency on retries.
- **Cancel / void**: Release an authorization or cancel a still-cancellable payment so no stray capture happens and no ambiguous "half-open" state sits between the order system and the rail.
- **Refund**: Return money after a successful collection with traceable linkage to the original payment; reliability depends on partial refunds, cutoffs, and consistent final states.
- **Status check (query / retrieve)**: Read the current state (`Pending`, `Authorized`, `Captured`, `Failed`, `Canceled`, `Refunded`, …) when the UI session, webhooks, or clocks do not give a single synchronous answer — essential whenever phases complete **asynchronously**.
- **Tokenize / save payment credential**: Create and store a reusable token (or saved instrument id) without retaining raw credential data; reliability requires clear scope, lifecycle controls, and deterministic token-to-instrument linkage.
- **Charge with saved token (CIT/MIT)**: Reuse a stored token for subsequent payments, with explicit initiator classification (**customer-initiated** vs **merchant-initiated**) and correct category indicators (for example recurring or unscheduled) to avoid issuer ambiguity.
- **Stored agreement / mandate lifecycle**: Record, retrieve, update, and revoke shopper consent (or mandate) for repeat charging, so token reuse remains auditable and compliant across renewals, retries, and cancellations.

## What’s Next
In upcoming blogs, we will:
- Define a complete payment capabilities map.
- Compare how leading PSP APIs support each capability.
- Show how this model evolves from payment functions to payment products and payment services.
