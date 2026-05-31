---
draft: true
title: Settlement Report Processing Engine (2) - The Vision
slug: settlement-report-processing-engine-2
date: 2026-05-31T10:00:00+0800
categories:
- Payments
tags:
- Settlement
- Reconciliation
---
The [previous post](/blogs/settlement-report-processing-engine-1/) validated the problem: reconciliation across PSPs breaks down in ingest, matching, and visibility before downstream teams can close with confidence. This post defines the engine vision — what a **settlement report processing engine** must deliver, which capabilities ship in the first release, and which features can follow once the core path is live.

## Product Vision

### Problem

Publisher settlement reports arrive in incompatible shapes, through different channels, on unpredictable schedules. Downstream teams stitch them together with scripts and spreadsheets because no dedicated engine owns the path from raw file to reconcilable output.

### Product

A **settlement report processing engine** (**engine**) that connects **publishers** — PSPs, acquirers, wallet operators, banks — to **consumers** — finance reconciliation jobs and internal data pipelines. The engine owns ingest, normalization, matching, reconciliation, and export. Close scheduling, approval, and reporting workflows stay with consumers.

### Design goals

The engine should be reliable at ingest, scale with volume peaks and publisher growth, trace every row to its source file and processing run, preserve auditable history through amendments, and onboard new report types through configuration rather than custom builds.

### Jobs to be done

For each settlement window and each merchant or MID, consumers must be able to answer three questions from engine output — not from ad hoc spreadsheets:

- Which internal transactions settled in this payout?
- What fees, reserves, and FX adjustments explain the delta between gross and net?
- Where settlement lines, payment records, and bank statement entries disagree, and what explains the exceptions?

### Success criteria

The first release delivers canonical settlement output consumers can query and export. Fee and gross-to-net questions resolve from normalized settlement lines; joins to payment records and bank entries ship in P1.

- **Unified pipeline**: One ingest-to-export path for every publisher, channel, and report type.
- **Canonical reconciliation model**: One settlement schema and one exception taxonomy regardless of publisher layout.
- **Delivery visibility**: Missing, late, or corrupt files surface before finance finds them manually.
- **Partial-failure tolerance**: Invalid rows quarantined; valid rows in the same file still commit.
- **Auditable history**: Raw artifacts and normalized rows retained in the engine; amendments replay without depending on publisher republication.
- **Consumer-shaped output**: Query and export over canonical fields; layout, grouping, and format chosen by the consumer.

## P0 Features

These capabilities implement the success criteria. Without them, the engine does not replace the fragmented pipelines.

- **Multi-channel ingestion**: Fetch from SFTP, email, API, and publisher portal; register expected delivery per publisher, report type, and settlement window.
- **Credential management**: Centralize SFTP keys, API tokens, and mailbox credentials; rotation and per-environment isolation.
- **Publisher onboarding**: Declarative configuration for publisher, report type, channel, and schedule — no bespoke pipeline per integration.
- **Configuration-driven formats**: Versioned definitions map CSV, fixed-width, XML, and JSON into the canonical model.
- **Amount normalization**: One sign convention and line-category semantics for gross, net, fee, tax, and transaction vs settlement currency.
- **Partial failure handling**: Row-level validation; quarantine failures with reason; commit valid rows independently.
- **Report scope and correlation**: Group files by payout batch, MID, currency, and legal entity into settlement windows.
- **Versioning and amendments**: Idempotent ingest; retain superseded artifacts; re-run downstream stages on adjustment journal lines.
- **Persistence and query**: Raw and normalized data in the engine per configured retention policy; query by merchant scope, window, line category, and quarantine reason.
- **Consumer APIs and exports**: Query, webhooks, and scheduled or on-demand batch extract — column layout, grouping, and format.
- **Volume at scale**: Stream or chunk large files; scope commits to batch and window; idempotent replay across concurrent T+1 deliveries.
- **Observability and audit**: Ingest lag and quarantine rate for operators; row lineage to source artifact, mapping version, and processing run.

## P1 Features

These follow the core ingest path. Matching and reconciliation need feeds the settlement report alone does not carry — internal payment records, bank statement entries, and a shared external id map; simulation and operator tooling reduce rollout risk once that path is stable.

- **Identifier matching**: Join settlement lines to payment records through an external id map — orchestrator ids, PSP references, auth codes, network RRN; ambiguous matches to review.
- **Three-way reconciliation**: Match settlement lines to payment records and bank statement entries; gross-to-net residuals that fee and reserve lines should explain.
- **Exception management**: Reason-coded classification for unmatched and mismatched lines; persisted match status and exception metadata for triage.
- **Matching control**: Start or monitor match runs, record operator decisions, and surface match rate and exception volume through the consumer API.
- **Settlement file simulation**: Test ingest and reconciliation against publisher-shaped fixtures before live files arrive.
- **AI-assisted operations**: Surface ingest health, mapping review, and quarantine triage in operator tooling — less spreadsheet work, same audit controls.
