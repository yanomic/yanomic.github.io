---
draft: true
title: Payment APIs — cross-cutting conventions
slug: payment-api-cross-cutting-conventions
date: 2026-05-13T09:00:00+0800
categories:
- Payments
tags:
- API Design
- Payment Capabilities
---

## Introduction

Payment orchestration APIs share the same transport and contract problems as any REST surface: the caller must retry safely, authenticate, parse failures consistently, and verify inbound webhooks. Capability posts (for example [Authorize, capture, cancel, and refund](/blogs/payment-capability-authorize-capture-cancel/)) stay focused on **resources and state**; this post records **cross-cutting** rules that apply to **every** endpoint on the API.

## Idempotency and client request ids

**Mutating** `POST` calls (creates, completions, increments, captures, cancels) should accept an optional **client request id** per logical operation — a stable string the merchant generates once per intent (`authorizationRequestId`, `completeRequestId`, `incrementRequestId`, `captureRequestId`, `cancelRequestId` in the reference payment surface).

- **When present:** the provider stores the key with the outcome of the first successful execution. A **replay** with the same key and compatible body returns the **same** response representation as the original (including the same resource ids). The provider does not run the side effect twice.
- **When omitted:** the caller gets ordinary HTTP semantics only. Retries after timeouts or client disconnects may duplicate work unless the merchant enforces uniqueness elsewhere (for example a conditional create on `merchantReference`, or out-of-band deduplication).
- **Scope:** keys are scoped **per operation type** and usually **per merchant credential** — the same string may not be reused across unrelated operation classes unless the API documents that explicitly.
- **Fresh keys for new intents:** partial capture, partial cancel, and similar shapes need a **new** request id for each new attempt; reuse a value only to **retry** the same attempt after an inconclusive HTTP result.

Some implementations expose the same idea as an **`Idempotency-Key`** header instead of a body field; the contract is identical.

## Transport encryption and key pairs

**Channel confidentiality and integrity** belong in **TLS** between client and API (TLS 1.2+ in practice; prefer TLS 1.3). The HTTP JSON body is not separately “encrypted with a key pair” on every request in typical REST designs — the **session keys** inside TLS handle bulk encryption; the **server certificate’s** public key bootstraps trust via the PKIX chain the client validates.

**Asymmetric key pairs** still matter at the API boundary for:

- **Webhook authentication:** the provider signs each delivery (often **HMAC** over the body with a shared secret, or **RSA/ECDSA** signatures with a published public key). The merchant verifies with the paired secret or public key — that is **authentication and integrity**, not general payload encryption.
- **Request signing (optional):** some B2B APIs require the client to sign a canonical string (method, path, timestamp, body hash) with a **merchant private key**; the API verifies with the registered public key. Again: integrity and origin proof, not a replacement for TLS.
- **Field-level or envelope encryption (rare on REST):** when a scheme demands hiding specific fields from the PSP, **JWE** or tokenization applies — that is a **separate** design from “REST over TLS” and belongs in the tokenization or PCI scope document, not the default payment JSON surface.

**Mutual TLS (mTLS)** is a common pattern for high-trust machine-to-machine integrations: both sides present certificates. It complements OAuth or API keys; it does not remove the need for application-layer authorization.

## Client and server authentication

- **Merchant to API:** API keys, **OAuth 2.0** client credentials, signed JWTs, or mTLS — pick one primary model and document how keys rotate.
- **API to merchant (webhooks):** signed payloads plus HTTPS on the merchant endpoint; optional IP allowlists are brittle alone.

## Versioning and compatibility

- **URL prefix** (`/v1/...`) or explicit **Accept** header versioning — choose one primary lever and freeze breaking changes behind a new version.
- **Additive** JSON fields are backward compatible; removing or retyping fields is not.

## Errors and correlation

- **Structured errors:** machine-readable `code`, human `message`, optional `details` with JSON paths for validation failures (`422`), stable `type` URI where useful.
- **HTTP mapping:** use `409` for state conflicts, `422` for semantic validation, `429` for rate limits with `Retry-After`, `503` for transient overload with safe retry semantics.
- **Correlation:** support **`X-Request-Id`** (or equivalent) end-to-end — the merchant supplies or the API mints one; return it on the response and include it in logs and support tooling.

## Pagination and listing

- Cursor-based lists for high-volume collections (`captures`, `cancels`, settlement lines); document default sort and maximum page size.
- Prefer opaque cursors over offset pagination for append-heavy ledgers.

## Timeouts, clocks, and rate limits

- Document client-side timeouts that still allow the server to finish (especially for auth and capture).
- All monetary and time fields need explicit **IANA timezone or UTC** rules and **ISO 8601** profile.
- Rate limits: publish limits and backoff guidance; use `429` + `Retry-After` consistently.

## Webhooks

- **HTTPS only** on the merchant URL; **HMAC** (or asymmetric signature) on the raw body; constant-time compare on verification.
- **At-least-once delivery:** duplicate events are normal; consumers dedupe on `(resourceId, eventType, terminalState)` or a monotonic `eventId`.
- **Acknowledgement contract:** `200` with a documented body; retries with exponential backoff and a replay window the dashboard exposes.

## Summary

Treat **TLS** as the default confidentiality layer; use **asymmetric or shared-secret signing** where the trust boundary is “untrusted network caller” (webhooks, optional request signing). Use **optional client request ids** on mutating `POST`s so retries after ambiguous HTTP outcomes converge. Keep capability posts about **state and resources**; keep this checklist for **everything that repeats on every route**.
