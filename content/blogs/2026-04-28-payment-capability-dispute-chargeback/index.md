---
draft: true
title: Payment Capability - Dispute and Chargeback
slug: payment-capability-dispute-chargeback
date: 2026-04-28T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Disputes
---

- **Semantics** — respond to shopper-initiated challenge on a captured/settled charge: receive cases, submit evidence, accept liability, and track outcomes under rail/scheme rules.
- **State model** — `CaseOpened` -> `EvidenceRequested` -> (`EvidenceSubmitted` -> `AwaitingArbitration`) -> (`Won` | `Lost` | `Accepted`), with LPM variants often simplified.
- **Recovery** — idempotent evidence submission by dispute id; deadline misses must hard-fail.
- **Time discipline** — strict representment/arbitration windows, typically reason-code specific.
- **Observability** — webhook per case transition plus dispute query (reason code, deadline, evidence state, expected outcome) and monthly dispute report reconciliation.
