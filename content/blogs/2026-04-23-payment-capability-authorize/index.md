---
draft: true
title: Payment Capability - Authorize, Capture, Cancel, and Refund
slug: payment-capability-authorize-capture-cancel
date: 2026-04-23T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Authorize
- Capture
- Cancel
- Refund
---

## Introduction

Authorization is the most consequential capability in the lifecycle. It **marks the start of a payment**, and it is where the most parties interact: shopper, merchant, PSP, acquirer, scheme or rail operator, issuer, and — on many LPMs — a wallet or platform sitting alongside the bank. Every substantive check happens inside this one call: **risk scoring**, **authentication of the payer**, **fraud screening**, **funds / balance check**, **velocity and regulatory limits**, **sanctions / AML**. A payment is only authorized once all of these pass.

## Single-Message vs Dual-Message 

Authorization terminates differently depending on whether authorization and clearing travel as **one message** or **two** — a foundational distinction that the rest of this post leans on. On **dual-message (DMS)** rails, a successful authorization is only a **hold** until **capture** presents the transaction into clearing for the amount the merchant is owed; on **single-message (SMS)** and SMS-equivalent rails, **one instruction** reaches **`Authorised`** and **`Captured`** together — there is no hold and no separate merchant capture call. The card industry names the split directly:

- **Single-Message System (SMS)**: also called `sale` or `purchase` — folds authorization and clearing into a single message, and there is no separate capture instruction. When authorization succeeds, the money is already moving.
- **Dual-Message System (DMS)**: `auth + capture`, sometimes labelled **two-step** in PSP documentation — reserves funds first and requires a separate capture call to push the transaction into clearing. Authorization leaves an **authorization hold** that the merchant later captures or releases.

Most card credit traffic is DMS; PIN debit and many regional debit schemes default to SMS.

LPMs do not share a formal label, but the same split shows up in their flows. The majority — **PIX**, **iDEAL**, **Bancontact**, **UPI**, **Swish**, **BLIK**, **Alipay**, **WeChat Pay** — are **SMS-equivalent**: when the rail confirms the shopper's action, funds are already moving and the Capture capability is a no-op. **Vouchers and offline-confirmation LPMs** — **Boleto**, **OXXO**, **Konbini**, **cash on delivery** — are **DMS-equivalent**: authorization issues a reference, and a separate confirmation (a capture call, or in some cases a settlement-side credit notice) lands once the shopper completes the off-rail step.

This split governs which downstream capabilities apply: **Capture** is required on DMS rails and a no-op on SMS; **Cancel** exists only on DMS (it releases an authorization hold that SMS rails never create); **Refund** exists everywhere.


## Authorization

The shape of the authorization call varies by rail and channel: a frictionless card flow closes in a single API round-trip, while a 3DS challenge, a redirect, a wallet handoff, or a voucher needs a follow-up call to submit the shopper-return data — 3DS challenge result, redirect callback, wallet token, voucher reference — before the rail returns a final state. Treat **initiate** and **complete** as two distinct touchpoints even when a given flow collapses them into one; Appendix A maps how each PSP names them.

### Card Flows

On cards, three shapes dominate:

- **Challenge flow** — the issuer requires an explicit interaction from the shopper to authenticate. The shopper is taken (inline iframe or full redirect) to the issuer's 3-D Secure ACS page and completes a challenge: OTP, bank-app push, biometric confirmation inside the banking app, or password. When the challenge finishes, the browser returns to the merchant and the merchant finalizes authorization with the authentication evidence (CAVV/ECI) attached.
- **Frictionless flow** — the issuer authenticates the shopper from device, transaction, and behavioral data alone and authorizes without any shopper interaction. 3-D Secure still runs (authentication evidence is still produced), but the challenge step is skipped and the merchant never leaves its own surface.
- **Non-3DS / SCA-exempt** — issuer authorises directly from the auth message; common in markets without an SCA mandate, for low-value or merchant-initiated transactions, and under regulator-approved exemptions (TRA, low-value, allowlist).

Either way, the merchant gets a **synchronous, final authorization outcome** once any authentication and the auth message complete — `Authorised`, `Declined`, or `Error`, with no fourth bucket called *"we'll let you know in a few minutes."* The whole chain runs on tight, scheme-enforced clocks: the issuer answers the auth message in seconds (and if it doesn't, the scheme returns a **stand-in** decision in its place), and the 3-D Secure challenge itself is bounded to a few minutes by the ACS. The only pending-shaped edge cases are recovery scenarios — the shopper's browser doesn't return cleanly from the ACS, or the merchant times out mid-call — and even those resolve within minutes via **status query** or **auth reversal**, not a multi-hour wait.

### LPM Flows

Where cards converge on the two shapes above, LPMs are far more diverse — they were designed to fit local markets and user behavior rather than a single rail. The channels worth distinguishing:

- **Browser redirect** to the bank or wallet login (the closest analog to card 3DS).
- **QR code** via bank or wallet app.
- **Deeplink / universal link** handoff from merchant app to bank/wallet app.
- **In-app / super-app context** with no handoff.
- **Push notification** approval in bank/wallet app.
- **SMS** approval or OTP.
- **Embedded code entry** (for example BLIK code).
- **Voucher / pay-by-code** flows completed later in store/ATM/online banking.
- **Bank transfer with reference** pushed by the shopper.
- **Manual / offline confirmation** arriving minutes to days later.

The common property across all of these channels is that **the shopper leaves the merchant's environment** to complete the payment. Nobody in the processing chain has direct visibility into what the shopper is doing until the rail reports back. The authorization result is therefore **asynchronous by nature** — not a special case, the default. Reliable integrations depend on **status queries**, **webhooks**, or both, and must not conflate "API returned" with "payment succeeded."

### Authorization State Machine

Because authorization can be async and the shopper isn't always reachable, every robust integration treats an authorization as a stateful resource with a bounded **expiry**.

- **`Received`** — submitted; expiry clock running.
- **`Authorised`** — rail approved before expiry. Success outcome for authorization; on DMS the open hold and capturable balance carry into capture below.
- **`Declined`** — refused by a party in the chain before expiry.
- **`Error`** — technical failure prevented a clean outcome.
- **`Expired`** — deadline elapsed with no final outcome.
- **`Closed`** — shopper explicitly aborted.

```mermaid
stateDiagram-v2
    [*] --> Received: authorization initiated
    Received --> Authorised: rail confirms
    Received --> Declined: rail / risk refuses
    Received --> Error: technical failure
    Received --> Expired: deadline elapsed
    Received --> Closed: shopper aborts
    Declined --> [*]
    Error --> [*]
    Expired --> [*]
    Closed --> [*]
    Authorised --> Captured: SMS-equivalent (same instruction)
```

On DMS rails, `Authorised` is not an authorization exit: it is the handoff into capture (below). The authorization capability still treats `Declined`, `Error`, `Expired`, and `Closed` as its failure finals. On SMS-equivalent rails, `Captured` follows in the same transition — the merchant may see one PSP status or two named states that land together.

## Capture

Capture is the instruction that turns a DMS authorization into money the merchant will actually receive. It is **required on DMS rails** and **collapsed into authorization on SMS-equivalent rails** — the merchant may still see `Authorised` and `Captured` on the same payment object, but there was no separate clearing instruction to issue.

### Individual vs batch capture

- **Individual capture** — a capture call per authorization when the fulfillment obligation is met.
- **Batch capture** — authorizations accumulated and submitted at the acquirer **clearing cutoff**.

Many PSPs expose a hybrid: per-authorization API calls while aggregating into a batch at cutoff.

### Capture state machine

Capture is **asynchronous on DMS rails**: the API accepts the instruction before clearing commits. Entry is always from **`Authorised`** on the parent authorization. PSPs differ on how much of that asynchrony is visible in the first HTTP response (some return `202` or `received` and complete in a webhook; see Appendix A).

> **Authorization lifetime vs capture window:**
> - **Stripe:** uncaptured Payment Intents are cancelled after a configured period by default ([Capture](https://docs.stripe.com/api/payment_intents/capture) intro).
> - **Checkout.com:** if `capture` is `false`, uncaptured payments are voided after seven days unless captured ([Capture a payment](https://www.checkout.com/docs/payments/manage-payments/capture-a-payment)). Treat these as **expiry** clocks separate from the **capture window** the scheme sets for presenting into clearing.

- **`CaptureReceived`** — accepted and in flight toward clearing.
- **`Captured`** — clearing accepted the capture.
- **`CaptureFailed`** — clearing refused the capture.
- **`CaptureError`** — technical failure prevented clean delivery.

```mermaid
stateDiagram-v2
    state Authorised
    Authorised --> CaptureReceived: capture initiated (individual or batched)
    Authorised --> Captured: SMS-equivalent (same instruction)
    CaptureReceived --> Captured: clearing accepts
    CaptureReceived --> CaptureFailed: clearing refuses
    CaptureReceived --> CaptureError: technical failure
    Captured --> [*]
    CaptureFailed --> [*]
    CaptureError --> [*]
```

On SMS-equivalent rails, `Authorised` and `Captured` are both valid states; they land in the same transition and skip `CaptureReceived`.

### Capture variants

Capture supports amount shapes beyond full one-shot capture:

- **Partial capture** — capture less than the authorized amount; the remainder stays on hold until captured or released.
- **Multiple captures** — several captures against the same authorization up to the authorized total.
- **Incremental authorization** — raise the authorized amount before capture when the order grows; each increment is an authorization-side change, not a capture state.
- **Over-capture** — allowed only in constrained scheme / MCC contexts.
- **Force capture** — capture without a matching live authorization; high-risk and restricted.

None of these shapes need new top-level payment states on the parent authorization.

- **Partial capture** and **multiple captures** — each attempt still runs `CaptureReceived` → `Captured` / `CaptureFailed` / `CaptureError`. The parent stays `Authorised` and carries `amountAuthorized`, `amountCaptured`, and `amountCapturable` until the cap is reached or the remainder is released or canceled. A partial capture is a `Captured` child below the authorization total; another shipment is another capture from `Authorised` until `amountCapturable` is zero.
- **Incremental authorization** — authorization-side only. The parent stays `Authorised` while `amountAuthorized` and `amountCapturable` rise; capture children do not move until a capture call.
- **Over-capture** — a capture attempt for more than `amountCapturable` but within scheme and MCC limits. The child still ends in `Captured` or `CaptureFailed`; the parent gains no dedicated over-capture status.
- **Force capture** — a capture with no matching live `Authorised` hold on that payment. The attempt enters `CaptureReceived` or fails as `CaptureFailed` / `CaptureError` when the rail requires an authorization reference.
- **Individual** and **batch** submission — timing only. Each attempt uses the same per-capture transitions; batching queues clearing until the acquirer **clearing cutoff**.

On SMS-equivalent rails, these variants are usually represented as separate transactions or refunds rather than capture calls against a hold.

## Cancel

Cancel is the instruction that releases an **active authorization hold** before clearing has accepted capture. It only applies on **DMS** rails. On **SMS** and SMS-equivalent rails the money has already moved by the time authorization succeeds, so there is no hold to release; the equivalent business outcome is a **refund**.

Cancel **moves no money**. The hold dissolves at the rail and the shopper's available credit or balance is restored once the issuer processes the reversal.

### Cancel variants

- **Full cancel** — release the **entire remaining** hold in one instruction (omit amount on the cancel call when the API treats that as “all of `amountCapturable`”).
- **Partial cancel** — a **partial reversal**: release part of the hold while the remainder stays live for capture. Not every acquirer or PSP exposes it.
- **Multiple partial cancels** — stack against the same authorization until the remaining authorized balance reaches zero or the merchant captures what is left.

Each attempt still runs `CancelReceived` → `Canceled` / `CancelFailed` / `CancelError`; the parent carries `amountAuthorized`, `amountCapturable`, and `PartiallyCanceled` / `Canceled` as in the state machine below. Canceling what remains **after** a partial capture is the same shape: cancel against the **remaining** hold, not a separate variant.

> **Notes on cancel variants**
> - **PSP vocabulary:** Many products say **void** or **reversal** instead of cancel; timing relative to batch still lands in the same per-operation states.
> - **LPM partial cancel:** LPMs essentially never support partial cancel. The few rails that expose any cancel usually expose only full-amount cancel.


### Cancel state machine

Cancel is often **fast at the HTTP boundary** because no capture has entered clearing yet, but some PSPs still return an **accepted** response (`received`, `202`) and deliver the rail outcome asynchronously in a **webhook** (for example Adyen **CANCELLATION** after [`POST /payments/{paymentPspReference}/cancels`](https://docs.adyen.com/api-explorer/Checkout/71/post/payments/(paymentPspReference)/cancels)).

- **`CancelReceived`** — accepted and in flight toward the rail.
- **`Canceled`** — accepted by the rail.
- **`CancelFailed`** — refused by the rail.
- **`CancelError`** — technical failure prevented a clean outcome.

```mermaid
stateDiagram-v2
    [*] --> CancelReceived: cancel submitted
    CancelReceived --> Canceled: rail accepts
    CancelReceived --> CancelFailed: rail refuses
    CancelReceived --> CancelError: technical failure
    Canceled --> [*]
    CancelFailed --> [*]
    CancelError --> [*]
```

The parent authorization carries aggregate state separately. Each successful per-operation `Canceled` decrements remaining authorized balance; the parent moves `Authorised` → `PartiallyCanceled` → `Canceled` as the hold is released in part or in full.

### Cancel after capture is initiated

If the merchant tries to cancel after capture is in flight:

- **`CaptureReceived` (pre-cutoff)** — the capture can often still be pulled from a pending batch.
- **`Captured` (post-cutoff)** — clearing already accepted; cancel fails and the merchant must **refund**.

If capture and cancel race for the same payment resource (`paymentId`), the processor orders them deterministically; both operations must be idempotent.

## Refund

Refund returns money to the shopper after clearing has accepted capture, linked to the original payment. It is the universal undo primitive across cards and most LPMs, with method-specific exceptions (some voucher and cash flows do not support automated reversal the same way).

Unlike cancel, refund **moves money** and appears as a separate credit transaction to the shopper.

### Partial and multiple refunds

Partial refund is widely supported. Multiple partial refunds can stack against the same capture with cumulative sum capped at the captured amount.

### Refund state machine

Refund is often synchronous at API acceptance, while cardholder-visible posting remains asynchronous.

- **`RefundReceived`** — accepted and in flight toward rail settlement.
- **`Refunded`** — accepted by the rail.
- **`RefundFailed`** — refused by the rail.
- **`RefundError`** — technical failure prevented a clean outcome.

```mermaid
stateDiagram-v2
    [*] --> RefundReceived: refund submitted
    RefundReceived --> Refunded: rail accepts
    RefundReceived --> RefundFailed: rail refuses
    RefundReceived --> RefundError: technical failure
    Refunded --> [*]
    RefundFailed --> [*]
    RefundError --> [*]
```

The parent payment tracks aggregate refund status separately (`PartiallyRefunded` → `Refunded`) alongside captured and refunded amounts.


## Endpoints

A capability-centred surface aligned with the state machines above. Authorization, capture, cancel, and refund share `paymentId`; each capture attempt gets `captureId`, each cancel attempt gets `cancelId`, and each refund attempt gets `refundId`. 

> Optional per-operation request ids on mutating `POST`s, replay behaviour, TLS, signing, errors, and webhooks follow [Payment APIs — cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/).

### Authorization

- **Initiate** (`POST /payments`): creates a payment resource and starts authorization.
    - **Request**: optional `authorizationRequestId` (client request id for this create); `amount`, `currency`, `merchantReference`, `paymentMethod` (or token reference), `channel`, `returnUrl`, optional `captureMode` (`AUTO` for SMS-equivalent rails, `MANUAL` for DMS holds), and optional merchant-set `expiresAt`.
    - **Response**: the payment resource — `paymentId`, `state` `Received`, `expiresAt`, `amountAuthorized`, `amountCaptured`, `amountCapturable`, and optional `nextAction` when the rail needs shopper interaction (`type`, redirect or scheme URL, app link, or 3DS challenge data, each with `expiresAt`).
    - **Errors**: validation failures return `422 Unprocessable Entity` with structured paths. Without a request id, repeated `POST /payments` may create duplicate resources unless the merchant enforces uniqueness elsewhere (for example on `merchantReference`).
- **Complete** (`POST /payments/{paymentId}/complete`): submits shopper-return data after redirect, 3DS, wallet, or voucher steps.
    - **Request**: `paymentId` in the path; optional `completeRequestId` (client request id for this completion); and a rail-specific completion payload — 3DS challenge result, redirect callback fields, wallet token, or voucher confirmation reference.
    - **Response**: the payment resource in its post-completion `state` (`Authorised`, `Declined`, `Error`, `Expired`, or `Closed`).
    - **Errors**: calling complete when the resource is already final returns `409 Conflict`. Calling complete when no `nextAction` was outstanding is a no-op with the current resource.
- **Increment** (`POST /payments/{paymentId}/increments`): raises `amountAuthorized` before capture when scheme and product allow incremental authorization.
    - **Request**: optional `incrementRequestId` (client request id for this increment) and the additional `amount` to add to the hold.
    - **Response**: the payment resource with `state` `Authorised` and updated `amountAuthorized` and `amountCapturable`.
    - **Errors**: calling increment when the parent is not `Authorised`, or when the rail rejects the raise, returns `409 Conflict` or `422 Unprocessable Entity` with a structured reason.
- **Status** (`GET /payments/{paymentId}`): reads the payment resource at any point in the lifecycle.
    - **Response**: `paymentId`, `state` (`Received`, `Authorised`, `PartiallyCanceled`, `Canceled`, `PartiallyRefunded`, `Refunded`, `Declined`, `Error`, `Expired`, `Closed`), `expiresAt`, `amountAuthorized`, `amountCaptured`, `amountCapturable`, refundable balance fields where applicable, structured decline or error reasons, and rail identifiers (`pspReference`, `authCode`, `cavv`, `eci`). The provider must answer this read before any webhook has fired.
- **Webhook**: outbound `POST` to the merchant's registered URL on each final authorization transition.
    - **Payload**: event type (`authorization.received`, `authorization.authorised`, `authorization.declined`, `authorization.expired`, `authorization.closed`, `authorization.error`) and the same payment resource shape as **Status**. Idempotent on `(paymentId, state)`.
    - **Acknowledgement**: per the cross-cutting conventions (signed delivery, `200 OK`, replay window).

### Capture

DMS only; SMS-equivalent rails omit separate capture calls.

- **Initiate capture** (`POST /payments/{paymentId}/captures`): presents part or all of an `Authorised` hold into clearing.
    - **Request**: optional `captureRequestId` (client request id for this capture) and `amount` ≤ `amountCapturable`.
    - **Response**: a capture resource — `captureId`, parent `paymentId`, `amount`, and `state` `CaptureReceived` when clearing is async, or `Captured` when the rail answers inline. Partial and multiple capture are repeated `POST`s until `amountCapturable` is zero or the hold is released.
    - **Errors**: capture when the parent is not `Authorised`, or for more than `amountCapturable` without an allowed over-capture path, returns `409 Conflict` or `422 Unprocessable Entity`.
- **Capture status** (`GET /payments/{paymentId}/captures/{captureId}`): reads one capture attempt.
    - **Response**: `captureId`, parent `paymentId`, `amount`, and `state` (`CaptureReceived`, `Captured`, `CaptureFailed`, `CaptureError`), plus clearing refusal or error detail when applicable.
- **List captures** (`GET /payments/{paymentId}/captures`): lists capture attempts on the parent authorization.
    - **Response**: an ordered list of capture resources for partial and multiple capture reconciliation. **Status** on the parent remains the aggregate view (`amountCaptured`, `amountCapturable`).
- **Webhook**: outbound `POST` per `captureId` when a capture attempt reaches a final state.
    - **Payload**: event type (`capture.received`, `capture.captured`, `capture.failed`, `capture.error`), `captureId`, `amount`, parent `paymentId`, and the capture resource fields from **Capture status**.
    - **Acknowledgement**: per the cross-cutting conventions.

### Cancel

DMS only; SMS-equivalent rails omit separate cancel calls.

- **Initiate cancel** (`POST /payments/{paymentId}/cancels`): releases part or all of an `Authorised` hold before clearing accepts capture.
    - **Request**: optional `cancelRequestId` (client request id for this cancel) and optional `amount` for partial cancel; omit `amount` for a full cancel of the remaining hold.
    - **Response**: a cancel resource — `cancelId`, parent `paymentId`, `amount`, and `state` `CancelReceived` when the rail answers async, or `Canceled` when the reversal lands inline. Partial cancel is repeated `POST`s until the parent reaches `Canceled` or the merchant captures the remainder.
    - **Errors**: cancel when the parent is not `Authorised` or `PartiallyCanceled`, when the amount exceeds the remaining hold, or when capture has already cleared, returns `409 Conflict` or `422 Unprocessable Entity`. A race with an in-flight capture resolves deterministically for that `paymentId`.
- **Cancel status** (`GET /payments/{paymentId}/cancels/{cancelId}`): reads one cancel attempt.
    - **Response**: `cancelId`, parent `paymentId`, `amount`, and `state` (`CancelReceived`, `Canceled`, `CancelFailed`, `CancelError`), plus rail refusal or error detail when applicable.
- **List cancels** (`GET /payments/{paymentId}/cancels`): lists cancel attempts on the parent authorization.
    - **Response**: an ordered list of cancel resources for partial cancel reconciliation. **Status** on the parent remains the aggregate view (`amountAuthorized`, `amountCapturable`, and parent `state` `Authorised`, `PartiallyCanceled`, or `Canceled`).
- **Webhook**: outbound `POST` per `cancelId` when a cancel attempt reaches a final state.
    - **Payload**: event type (`cancel.received`, `cancel.canceled`, `cancel.failed`, `cancel.error`), `cancelId`, `amount`, parent `paymentId`, and the cancel resource fields from **Cancel status**.
    - **Acknowledgement**: per the cross-cutting conventions.

### Refund

Applies once the parent has reached `Captured` (or equivalent settled state on the rail). Partial and multiple refunds are repeated `POST`s until the refundable balance is exhausted.

- **Initiate refund** (`POST /payments/{paymentId}/refunds`): returns part or all of a captured amount to the shopper.
    - **Request**: optional `refundRequestId` (client request id for this refund) and `amount` ≤ refundable balance on the payment (or on a specified `captureId` when the API scopes refunds to a capture).
    - **Response**: a refund resource — `refundId`, parent `paymentId`, optional `captureId`, `amount`, and `state` `RefundReceived` when settlement is async, or `Refunded` when the rail answers inline.
    - **Errors**: refund when the parent is not in a refundable state, for more than the refundable balance, or outside the rail refund window, returns `409 Conflict` or `422 Unprocessable Entity`.
- **Refund status** (`GET /payments/{paymentId}/refunds/{refundId}`): reads one refund attempt.
    - **Response**: `refundId`, parent `paymentId`, `amount`, and `state` (`RefundReceived`, `Refunded`, `RefundFailed`, `RefundError`), plus rail refusal or error detail when applicable.
- **List refunds** (`GET /payments/{paymentId}/refunds`): lists refund attempts on the parent payment.
    - **Response**: an ordered list of refund resources for partial and multiple refund reconciliation. **Status** on the parent remains the aggregate view (`PartiallyRefunded`, `Refunded`, and amounts).
- **Webhook**: outbound `POST` per `refundId` when a refund attempt reaches a final state.
    - **Payload**: event type (`refund.received`, `refund.refunded`, `refund.failed`, `refund.error`), `refundId`, `amount`, parent `paymentId`, and the refund resource fields from **Refund status**.
    - **Acknowledgement**: per the cross-cutting conventions.


## The Five Lenses

### Authorization

- **Semantics**: confirm payer intent and secure rail authorisation — a hold to capture later on DMS rails, or authorisation and clearing in one step on SMS-equivalent rails.
- **State model**: one in-flight state (`Received`) and four authorization failure finals (`Declined`, `Error`, `Expired`, `Closed`). `Authorised` is the success handoff into capture on DMS; on SMS-equivalent rails `Authorised` and `Captured` coalesce in one rail step.
- **Recovery**: optional client request ids on mutating `POST`s (see [cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/)); when the response or return path leaves the outcome unclear, query status, then reverse the auth if the rail still allows it.
- **Time discipline**: merchant-set expiry bounds the authorization; 3-D Secure, redirect, and voucher hold windows are shorter clocks inside that bound.
- **Observability**: many card rails answer synchronously; when they do not, or on async LPMs, webhooks and status queries are how the merchant learns the final outcome.

### Capture 

- **Semantics**: present a prior authorization into clearing; capture decides what amount is owed and when it enters the clearing stream.
- **State model**: begins at `Authorised`; on DMS, one in-flight capture state (`CaptureReceived`) and three capture finals (`Captured`, `CaptureFailed`, `CaptureError`); on SMS-equivalent rails, `Captured` follows `Authorised` in the same rail step without `CaptureReceived`. Partial and multiple captures reuse those states; the parent authorization tracks authorized, captured, and capturable amounts.
- **Recovery**: optional `captureRequestId` on capture `POST`s (see [cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/)); status query before replay when the HTTP outcome is uncertain.
- **Time discipline**: **capture window** and **clearing cutoff** are the controlling clocks; batch submission adds a second cutoff boundary.
- **Observability**: webhooks and status query per capture object; settlement reporting is the reconciliation source of truth.

### Cancel

- **Semantics**: release an active authorization hold before clearing has accepted capture; DMS-only; no money movement.
- **State model**: per-operation states (`CancelReceived`, `Canceled`, `CancelFailed`, `CancelError`); the parent authorization tracks `Authorised`, `PartiallyCanceled`, and `Canceled` as the hold is released in part or in full.
- **Recovery**: optional `cancelRequestId` on cancel `POST`s (see [cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/)); a race with capture must fail deterministically; recover `CancelError` via status query.
- **Time discipline**: bounded by clearing cutoff and authorization expiry.
- **Observability**: synchronous HTTP acceptance where the product returns a final body; otherwise webhooks and status query carry the rail outcome (see Appendix A for provider-specific capture and cancel notifications).

### Refund

- **Semantics**: return money after clearing acceptance; full, partial, and multiple refunds where the rail allows.
- **State model**: one in-flight state (`RefundReceived`) and three refund finals (`Refunded`, `RefundFailed`, `RefundError`); the parent payment tracks `PartiallyRefunded` and `Refunded` as credits land in part or in full, with amount fields tying back to captured balance.
- **Recovery**: optional `refundRequestId` on refund `POST`s (see [cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/)); server-side cap enforcement on cumulative partial refunds; status query before replay when the HTTP outcome is uncertain.
- **Time discipline**: rail-specific refund window plus settlement lag to cardholder-visible posting.
- **Observability**: webhooks, status query per refund object, and reconciliation report lines tied to `refundId`.

## Summary

Authorization is where a payment starts and where the chain runs its substantive checks: risk, payer authentication, fraud screening, funds availability, velocity and regulatory limits, and sanctions / AML. The rail-level split between single-message (authorization and clearing in one instruction) and dual-message (authorization hold, then capture) governs what happens next — capture is required on DMS rails and a no-op on SMS-equivalent rails; cancel exists only where a hold was created; refund applies everywhere. Cards are mostly DMS; many LPMs behave like SMS once the shopper completes the rail step, while voucher and offline-confirmation LPMs behave like DMS because confirmation arrives in a separate step.

On cards, challenge, frictionless, and non-3DS paths still end in a synchronous approved-or-declined outcome once authentication and the auth message complete, with scheme-enforced clocks and stand-in when the issuer does not answer in time. LPMs diverge by channel — redirect, QR, deeplink, push, SMS, embedded codes, vouchers, bank transfer with reference — but they share one property: the shopper leaves the merchant environment, so the authorization result is asynchronous by default and reliable integrations treat webhooks and status queries as part of the capability itself.

Model authorization as a stateful resource with a bounded expiry: one in-flight `Received` state, four authorization failure finals (`Declined`, `Error`, `Expired`, `Closed`), and `Authorised` as the success handoff. Recovery uses optional client request ids when the merchant needs replay safety (see [Payment APIs — cross-cutting conventions](/blogs/payment-api-cross-cutting-conventions/)), plus status query or auth reversal when the outcome is uncertain, and expiry as the primary time boundary across rail-specific sub-clocks. On DMS rails, `Authorised` opens the capture window and the cancel path: capture runs through `CaptureReceived` to `Captured`, `CaptureFailed`, or `CaptureError`; cancel runs through `CancelReceived` to `Canceled`, `CancelFailed`, or `CancelError`, with the parent moving to `PartiallyCanceled` or `Canceled` as the hold is released. Clearing cutoff and batch submission bound capture; cancel races with in-flight capture resolve deterministically for that `paymentId`. On SMS-equivalent rails, `Authorised` and `Captured` still apply and land in the same rail step — without `CaptureReceived`, a separate merchant capture call, or cancel against a hold. After `Captured`, refund is the money-moving reversal: each attempt runs `RefundReceived` to `Refunded`, `RefundFailed`, or `RefundError`, with the parent moving to `PartiallyRefunded` or `Refunded` as credits stack up to the captured total.

-----
## Appendix

### Appendix A: Provider docs (authorize, capture, cancel)

Links are the canonical API reference or guide entry points the post leans on. Naming differs by PSP (`capture` vs `settle`, `cancel` vs `void`); the capability model in the main text maps onto these operations.

- **Stripe (Payment Intents):** [Create](https://docs.stripe.com/api/payment_intents/create), [Confirm](https://docs.stripe.com/api/payment_intents/confirm), [Retrieve](https://docs.stripe.com/api/payment_intents/retrieve), [Capture](https://docs.stripe.com/api/payment_intents/capture), [Cancel](https://docs.stripe.com/api/payment_intents/cancel), [Verifying status](https://docs.stripe.com/payments/payment-intents/verifying-status). Cancel on `requires_capture` releases uncaptured funds (Stripe documents automatic refund of `amount_capturable`).
- **Adyen (Checkout API v71):** [`POST /payments`](https://docs.adyen.com/api-explorer/Checkout/71/post/payments), [`POST /payments/details`](https://docs.adyen.com/api-explorer/Checkout/71/post/payments/details), [`POST /sessions`](https://docs.adyen.com/api-explorer/Checkout/71/post/sessions), [`GET /sessions/{sessionId}`](https://docs.adyen.com/api-explorer/Checkout/71/get/sessions/(sessionId)), [`POST /payments/{paymentPspReference}/captures`](https://docs.adyen.com/api-explorer/Checkout/71/post/payments/(paymentPspReference)/captures), [`POST /payments/{paymentPspReference}/cancels`](https://docs.adyen.com/api-explorer/Checkout/71/post/payments/(paymentPspReference)/cancels) (outcome in **CAPTURE** / **CANCELLATION** webhooks; optional `Idempotency-Key` header on mutating calls), [Webhook `AUTHORISATION`](https://docs.adyen.com/api-explorer/Webhooks/1/post/AUTHORISATION).
- **Antom (AMS):** [`POST /v1/payments/pay`](https://docs.antom.com/ac/ams/payment_cashier), [`POST /v1/payments/capture`](https://docs.antom.com/ac/ams/capture), [`POST /v1/payments/cancel`](https://docs.antom.com/ac/ams/paymentc_online), [inquiryPayment](https://docs.antom.com/ac/ams/paymentri_online), [notifyPayment](https://docs.antom.com/ac/ams/paymentrn_online).
- **Airwallex (Payments):** [Create PaymentIntent](https://www.airwallex.com/docs/api/payments/payment_intents/create), [Confirm](https://www.airwallex.com/docs/api/payments/payment_intents/confirm), [Retrieve](https://www.airwallex.com/docs/api/payments/payment_intents/retrieve), [Capture](https://www.airwallex.com/docs/api/payments/payment_intents/capture), [Cancel](https://www.airwallex.com/docs/api/payments/payment_intents/cancel), [Webhooks](https://www.airwallex.com/docs/payments/reference/payments-webhooks), [Statuses](https://www.airwallex.com/docs/payments/reference/payment-statuses).
- **Checkout.com:** [Authorize a payment](https://www.checkout.com/docs/payments/manage-payments/authorize-a-payment), [Capture a payment](https://www.checkout.com/docs/payments/manage-payments/capture-a-payment) (`POST .../payments/{id}/captures`; `capture_type` for partial / multi-capture), [Void a payment](https://www.checkout.com/docs/payments/manage-payments/void-a-payment) (`POST .../payments/{id}/voids`), [API reference](https://api-reference.checkout.com/), [`authorization_approved` webhook](https://www.checkout.com/docs/developer-resources/webhooks/webhook-event-types/authorization_approved), [`payment_captured`](https://www.checkout.com/docs/developer-resources/event-notifications/event-types/payment_captured), [`payment_voided`](https://www.checkout.com/docs/developer-resources/event-notifications/event-types/payment_voided).
- **Worldpay (Access, Card Payments):** [Take a payment / authorize](https://developer.worldpay.com/access/products/card-payments/v6/authorize-a-payment), [Query a payment](https://developer.worldpay.com/products/card-payments/openapi/query-a-payment), [Manage payments](https://developer.worldpay.com/products/card-payments/openapi/manage-payments) (settle for capture; full and partial authorization cancellations), [Events](https://developer.worldpay.com/access/products/events/openapi).
- **Worldline (Connect):** [Create payment](https://apireference.connect.worldline-solutions.com/s2sapi/v1/en_US/java/payments/create.html), [Get payment](https://apireference.connect.worldline-solutions.com/s2sapi/v1/en_US/java/payments/get.html), [Capture payment](https://apireference.connect.worldline-solutions.com/s2sapi/v1/en_US/java/payments/capture.html), [Cancel payment](https://apireference.connect.worldline-solutions.com/s2sapi/v1/en_US/java/payments/cancel.html), [Webhooks](https://docs.connect.worldline-solutions.com/support/faq/connect/webhooks), [Statuses](https://docs.direct.worldline-solutions.com/en/integration/api-developer-guide/statuses).
