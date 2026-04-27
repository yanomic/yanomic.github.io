---
draft: true
title: Payment Capability - Settlement and Payout
slug: payment-capability-settlement-payout
date: 2026-04-29T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Settlement
---

- **Semantics** — merchant-observable money movement from acquirer/PSP to merchant bank account, net of fees, reserves, and adjustments.
- **State model** — payout lifecycle `Scheduled` -> `InTransit` -> (`Paid` | `Failed` | `Returned`).
- **Recovery** — payout retries controlled by provider; merchant needs bank account update and resend path tied to payout reference.
- **Time discipline** — contractual payout cadence plus reserve and FX clocks.
- **Observability** — payout transition events, payout query/list, and per-line payout breakdown linked to underlying transactions.

### Reconciliation and Reporting

- **Semantics** — authoritative record of money-moving events joinable to merchant ledger.
- **State model** — report lifecycle `Generating` -> `Available` -> `Superseded`.
- **Recovery** — idempotent report ingestion by report id, with stable external ids across transaction objects.
- **Time discipline** — generation lag, amendment window, and retention window.
- **Observability** — each report line should include identifiers, fees, FX fields, settlement currency, and linked payout id.
