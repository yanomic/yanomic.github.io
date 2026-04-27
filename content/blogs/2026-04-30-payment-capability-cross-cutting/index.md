---
draft: true
title: Payment Capability - Cross-Cutting
slug: payment-capability-cross-cutting
date: 2026-04-30T09:00:00+0800
categories:
- Payments
tags:
- Payment Capabilities
- Cross-Cutting
---

### Status Query

- **Semantics** — read the **current authoritative state** of any object (application, payment, refund, token, mandate, dispute, payout) by id. This is the capability merchants fall back to whenever the other channels disagree or go silent.
- **State model** — none of its own; it returns the state of other objects. What matters is **what fields it exposes**: not just the top-level status but the sub-states, timestamps, and linked object ids needed to make the state actionable.
- **Recovery** — the primary recovery primitive for the whole API surface. Every capability above implicitly assumes status query exists and is consistent with webhooks. If the two disagree, the query should be the tiebreaker.
- **Time discipline** — freshness matters more than latency. A query that returns data minutes stale during a settlement cutoff is worse than a query that is slower but current. Providers should document **maximum staleness**, not just p99 latency.
- **Observability** — the capability is itself observability, but the important meta-property is **completeness**: is there an object type you can receive a webhook for but not query? Any such asymmetry is a reliability bug.

### Webhooks / Event Stream

- **Semantics** — push state transitions from the provider to the merchant so polling is not the only path. Every lifecycle state transition must be expressible as an event.
- **State model** — events carry their own delivery state: `Pending` -> (`Delivered` | `Failed` -> `Retrying` -> `DeadLettered`). Dead-lettered events must be retrievable and replayable, or gaps in the merchant's view of the world become unrecoverable.
- **Recovery** — **signed**, **ordered where it matters**, **at-least-once delivered**, **replayable from a cursor**. Every consumer must be idempotent on the event id. Out-of-order delivery across objects is tolerable. Out-of-order delivery within a single object's lifecycle is not.
- **Time discipline** — delivery SLA, retry schedule, and retention window (how far back can I replay?) are the three clocks. A retention window shorter than the dispute window is a recurring source of data loss.
- **Observability** — the merchant must be able to inspect **failed deliveries**, **replay by id or time range**, and **filter by event type** without opening a support ticket.

### Status Query vs Webhook

> **TODO:** expand this comparison. Candidate structure:
>
> - When to use which, and why both are needed rather than one.
> - Latency, freshness, and completeness trade-offs.
> - Ordering, duplicates, and retries — who owns what.
> - Failure modes of each in isolation (silent webhook loss; stale query caches) and how the pair compensates.
> - A side-by-side table across: direction, cardinality, latency, ordering, replay, retention, auth model, and typical failure modes.
