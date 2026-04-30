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

Payment methods do not behave the same way. The split between cards and local payment methods (**LPMs**) is the obvious one, but each category hides its own variations.

This creates real friction on both sides. **Merchants** want to accept payments reliably. **Shoppers** want to pay with their preferred method. Both sides end up navigating inconsistent interactions, edge cases, and operational trade-offs they should not need to know about to complete a simple transaction.

This blog series focuses on **merchant-facing APIs** and treats **capabilities** in **functional** terms: only what crosses the integration boundary matters — inputs, responses, observable state, and end-to-end behavior — with the payment path modeled as a **black box**.

## Ecosystem and Lifecycle

Card payments are a useful reference because the roles are well defined and the same lifecycle vocabulary appears across many integrations. The ecosystem is not just "shopper and merchant"; several specialized parties cooperate, each with a narrow responsibility.

**Typical parties in a card transaction**

- **Cardholder**: The person whose card is used; proves identity and consent (3-D Secure, CVV, PIN where applicable) and holds the account at the **issuer**.
- **Merchant**: Sells goods or services and initiates payment requests toward its **acquirer** (directly or through a **gateway / PSP**).
- **Acquirer** (merchant bank): Underwrites the merchant, routes authorization and capture messages, receives settlement from the card network, and credits the merchant (minus fees). 
- **Scheme** (Card network): Visa, Mastercard, and others. Sets rules, routes messages between acquirer and issuer, and operates clearing and often parts of dispute handling. Networks do not hold the cardholder's money; they coordinate messaging and settlement between banks.
- **Issuer** (cardholder bank): Issues the card, approves or declines authorization based on risk and available funds, posts charges to the cardholder, and participates in clearing, settlement, and disputes on the cardholder side.
- **Gateway / PSP** (optional but common): A **gateway** is primarily the technical integration layer — API surface, request orchestration, tokenization, and routing to one or more acquirers. A **PSP** usually includes that gateway layer plus broader commercial and operational scope: merchant onboarding support, risk/fraud operations, reconciliation/reporting tooling, and sometimes settlement or payout services. From the merchant's perspective, this layer is often the primary integration surface even though settlement still runs through acquirer-network-issuer rails. In this series, we use gateway and PSP interchangeably for readability, unless a section needs the distinction explicitly.

With the parties in place, the same transaction flows through a sequence of well-defined phases. The sections below walk through each one in order.

### Onboarding
The merchant (optionally via PSP) onboards with an acquirer: KYC, pricing, MCC, and technical connectivity. The acquirer underwrites the merchant against **scheme rules**: PCI, branding, permitted category use, and cardholder-data handling. In the usual path the **acquirer** validates and approves the application (a **PSP** may handle operations in front of the acquirer). Direct scheme approval is the exception: some programs, high-risk categories, or network registration require the acquirer to file with the scheme, which then accepts or declines. **The cardholder is not involved**; onboarding only establishes what the merchant may initiate later at checkout.

```mermaid
sequenceDiagram
    participant Mer as Merchant / PSP
    participant Acq as Acquirer
    participant Sch as Scheme
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
The merchant (or automated rules) sends **capture** instructions for all or part of the authorized amount. The acquirer presents those transactions into **clearing**: the scheme exchanges clearing records with the issuer so the charge can be posted to the cardholder. Capture is about what is owed and **moving the transaction into clearing**; it is still distinct from **settlement**, where money actually moves between banks.

### Cancel
If the merchant will not capture — order canceled, inventory unavailable, or duplicate auth — they **cancel** the authorization while it is still valid. The acquirer asks the issuer to **release the hold**; no capture means no clearing/settlement for that authorization. Naming varies by provider (`void`, `cancel`, `reverse`); the idea is the same: end the hold without taking money.

### Refund
Refunds return money to the cardholder after a successful capture. Operationally, the merchant submits a refund against an existing cleared payment (full or partial), the acquirer validates amount and eligibility, and then sends a credit message through the scheme to the issuer. The issuer posts the credit to the cardholder statement, usually asynchronously from the API response, so merchant systems must track both acceptance and final posting; timing, cutoff behavior, and posting speed remain network- and issuer-dependent.

### Settlement
After clearing, **interbank settlement** nets obligations between issuer and acquirer according to the scheme's arrangements. Separately, the **acquirer settles to the merchant**: payout timing, reserves, and fees are defined in the merchant's contract. The scheme orchestrates settlement between issuer and acquirer; it does not replace the acquirer's merchant payout.

### Dispute
Unlike refunds, **disputes** start from the cardholder side: the **cardholder** challenges the charge with the **issuer**, which opens a **chargeback** (or similar) case. The acquirer and merchant then follow a scheme-defined evidence process with fixed timelines.

### Tokenization
The lifecycle above focuses on a **one-off** payment path. This section extends that model to **stored-instrument** journeys, where a payment credential is saved first and then referenced in later transactions. This extension is easiest to read as **two connected phases**: 

1. **Token creation**: in a **shopper-present** checkout, the payment credential is tokenized and saved for later reuse.
2. **Subsequent charge**: later authorizations reference that saved token, often as **shopper-not-present** transactions.


#### Phase 1: Token Creation

Phase 1 is a **customer-initiated transaction (CIT)**. Provisioning and consent are handled within the same checkout flow: the cardholder supplies card details, and the **gateway / PSP** requests tokenization. Depending on the model, the token is provisioned either by the PSP vault or by a **network** or **issuer** token service, then bound to the underlying card for later scheme routing. In that same CIT flow, the shopper accepts terms to save the card, completes **SCA** when required, and the authorization carries the needed **stored credential** indicators.

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

##### MIT Classification

These **business patterns** describe how **Phase 2** legs are labeled for schemes; they mostly apply when the merchant **initiates** the charge:

| Term | Meaning |
|------|--------|
| **Card on file (CoF)** | Umbrella: the card is **stored** (as a token) for later use. Does not by itself mean subscription. |
| **Subscription** | **Scheduled** charges (e.g. monthly fee): MIT with a **recurring** indicator; amount may be fixed or variable per your agreement. |
| **Unscheduled card on file (UCOF)** | Industry label for **merchant-initiated** charges **without** a fixed schedule — e.g. metered use, top-ups, “charge when balance low.” Mastercard uses “UCOF”; Visa’s aligned category is “Unscheduled Credential on File” (same acronym). Still requires valid prior **CIT** agreement from Phase 1. |

## Local Payment Methods

For local payment methods (LPMs), the **ecosystem and lifecycle are largely the same** as cards: similar participants and the same high-level phases — initiate, confirm, collect funds, reconcile, and handle refunds and problems. The **differences are in the details**: who plays each role, how authorization is triggered, and how refunds and disputes work.

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

Cards are not universally "better"; they are a **shared rail with predictable roles**. LPMs trade that uniformity for **local reach, lower cost in some markets, or shopper preference** — at the cost of more asynchronous states and method-specific operational playbooks. The next section defines **The Five-Lens Framework**: **functions** as the integration units, **five metrics** for comparing how PSP APIs expose them, and shared vocabulary for cross-method comparison.

## The Five-Lens Framework

The **Ecosystem and Lifecycle** section tells you *who* is involved and *how* work unfolds from onboarding through disputes. That story orients you in time and roles; it does not give you a blueprint for what to build. **Payment capabilities** are the guideline for what to implement — the merchant-facing operations and signals that make each phase executable, recoverable, and observable without guesswork or manual follow-up. This series treats each capability as a **function** and uses those functions as the organizing keys for the posts that follow.

**The five lenses** are the **metrics** for each function — not a parallel taxonomy, but the dimensions on which you measure semantics, failure behavior, clocks, and how state surfaces to the merchant when you line PSP APIs up against each other.

### The Five Lenses

We define the five metrics once below. What usually differs across rails is not whether a **function** exists, but **how it behaves** under each metric — including under retries, asynchronous flows, deadline pressure, and failure. Comparison is therefore not a race to match feature names; it is whether **that function** behaves consistently when you measure it with **the same five metrics**. That keeps comparisons fair while leaving room for rail-specific detail.

- **Semantics**: what the function does and does not do. The input/output contract, and the single question it answers.
- **State model**: the statuses the function produces or transitions through, and the allowed transitions between them. Which states are terminal, which are asynchronous, and where the next step lives.
- **Recovery**: how to stay correct under retries, timeouts, partial failures, duplicate webhooks, and out-of-order events. Idempotency anchors, reconciliation primitives, and safe fallbacks.
- **Time discipline**: the clocks and windows that govern the function: review SLAs, expirations, capture windows, representment deadlines, settlement lags.
- **Observability**: how the merchant learns the current state and the history behind it: synchronous responses, status queries, webhooks, reports, and reconciliation files.

## Closing

This chapter establishes **ecosystem and lifecycle** vocabulary — who participates and how work moves from onboarding through disputes — and frames **payment capabilities** as merchant-facing **functions** measured with **five metrics**.

The posts that follow move **phase by phase** through the lifecycle. Each installment applies those **metrics** to the **functions** that matter in that phase — comparing PSP API surfaces without flattening rail-specific detail.
