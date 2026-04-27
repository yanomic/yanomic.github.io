---
draft: true
title: Payment Capability - Tokenization
slug: payment-capability-tokenize
date: 2026-04-27T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Tokenization
---

## Tokenize

Tokenization in this chapter covers three tightly coupled surfaces: creating credential references (`Tokenize`), using them for repeat charges (`CIT/MIT`), and managing the shopper agreement or mandate lifecycle that authorizes those charges.

- **Semantics** — create and store a **reusable reference** to a payment credential so later charges can be issued without retaining raw credential data.
- **State model** — `TokenRequested` -> `Active` -> (`Suspended` | `Expired` | `Deleted`).
- **Recovery** — idempotent token provisioning reference; typed failures for graceful fallback behavior.
- **Time discipline** — token validity follows underlying instrument/mandate lifecycle and provider inactivity rules.
- **Observability** — token lifecycle webhooks plus list/query endpoint by shopper.

### Charge with Saved Token (CIT / MIT)

- **Semantics** — reuse a stored token for new transaction with explicit initiator class: **CIT** (customer initiated) vs **MIT** (merchant initiated).
- **State model** — same transaction state machine as first-time payment, plus required linkage to original CIT agreement.
- **Recovery** — idempotent transaction reference and MIT-specific decline handling to avoid unsafe retries.
- **Time discipline** — correct MIT category indicators (recurring/installment/unscheduled/reauthorization) on every charge.
- **Observability** — MIT transactions must also be observable as usage events of the underlying agreement.

### Mandate / Agreement Lifecycle

- **Semantics** — record, retrieve, update, and revoke shopper consent for repeat charging.
- **State model** — `Pending` -> `Active` -> (`Paused` | `Revoked` | `Expired`).
- **Recovery** — refresh merchant-side mandate view from rail/provider state; reconcile periodically where no push events exist.
- **Time discipline** — inactivity expiry, fixed-term expiry, and pre-notification windows become pre-charge constraints.
- **Observability** — mandate events webhook, mandate query/list, and joinability with transactions.
