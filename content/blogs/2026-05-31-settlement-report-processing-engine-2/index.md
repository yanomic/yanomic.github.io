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

A **settlement report processing engine** (**engine**) that connects **publishers** — PSPs, acquirers, wallet operators, banks — to **consumers** — finance reconciliation jobs and internal data pipelines. The engine owns ingest, normalization, matching, and export. Close scheduling, approval, and reporting workflows stay with consumers.

### Design goals

The engine should be reliable at ingest, scale with volume peaks and publisher growth, trace every row to its source file and processing run, preserve auditable history through amendments, and onboard new report types through configuration rather than custom builds.

### Jobs to be done

For each settlement window and each merchant or MID, consumers must be able to answer three questions from engine output — not from ad hoc spreadsheets:

- Which internal transactions settled in this payout?
- What fees, reserves, and FX adjustments explain the delta between gross and net?
- Where settlement lines, existing records, and bank statement entries disagree, and what explains the exceptions?

### Success criteria

The engine must enable consumers to answer those three questions confidently. It delivers:

- **One path across publishers**: The same ingest-to-export workflow for every report type and delivery channel.
- **Standardized reconciliation**: One canonical settlement schema and one exception taxonomy, regardless of publisher layout.
- **Proactive delivery tracking**: Expected files registered per publisher and window; alerts when artifacts are missing, late, or corrupt.
- **Resilient ingest**: Invalid rows quarantined; valid rows in the same file still processed.
- **Durable history**: Raw reports and normalized rows retained so reprocessing and audit do not depend on publisher republication.
- **Reliable joins**: Settlement lines matched to payment records and bank movements; ambiguous or unmatched lines routed to review — not silently attached.
- **Configurable extension**: New publishers, report types, and formats onboarded through configuration; export layout and schedule controlled by consumers without forking core logic.
- **Consumer-ready output**: Normalized rows, match outcomes, exceptions, and lineage available through query and export APIs.

## P0 Features

These map directly to the success criteria above. Together they are the minimum output consumers need to answer the three questions without spreadsheets. Without them, the engine does not replace the [fragmented pipelines](/blogs/settlement-report-processing-engine-1/).

- **Multi-channel ingestion**: Ingest from SFTP, email, API, and publisher portal; register expected delivery per publisher, report type, and settlement window; alert on missing, late, or corrupt files.
- **Credential management**: Centralize SFTP keys, API tokens, and mailbox credentials; support rotation and per-environment isolation.
- **Publisher onboarding**: Configure publisher, report type, channel, and schedule declaratively — no bespoke pipeline per integration.
- **Configuration-driven formats**: Map CSV, fixed-width, XML, and JSON layouts into one canonical model through versioned definitions.
- **Amount normalization**: Align per-publisher signs, line categories, gross/net/fee/tax semantics, and transaction vs settlement currency to one convention.
- **Partial failure handling**: Validate row by row; quarantine failures with reason; commit valid rows so one bad line does not block the file.
- **Report scope and correlation**: Group files by payout batch, MID, currency, and legal entity into settlement windows before matching.
- **Identifier matching**: Join through an external id map — orchestrator ids, PSP references, auth codes, network RRN — and route ambiguous matches to review.
- **Versioning and amendments**: Ingest idempotently; retain superseded artifacts; re-run downstream stages when adjustment journal lines arrive.
- **Three-way reconciliation**: Match settlement lines to payment records and bank statement entries; surface gross-to-net deltas fee and reserve lines should explain.
- **Exception management**: Classify unmatched and mismatched lines by reason code; persist match status and exception metadata; expose for downstream triage.
- **Persistence and query**: Retain raw and normalized data for the publisher retention window; query by merchant scope, window, match status, and exception reason.
- **Consumer APIs and exports**: Query, matching control, webhooks, and configurable batch output — column layout, grouping, format, and schedule.
- **Volume at scale**: Stream or chunk large files; scope commits to batch and window; support idempotent replay across concurrent T+1 deliveries.
- **Observability and audit**: Ingest lag, match rate, exception volume, and quarantine rate visible to operators; every row traceable to source artifact, mapping version, and processing run.

## P1 Features

These accelerate rollout and day-to-day operations. They are not required for the first production reconciliation path.

- **Settlement file simulation**: Test ingest and reconciliation against publisher-shaped fixtures before live files arrive.
- **AI-assisted operations**: Surface ingest health, mapping review, and quarantine triage in operator tooling — less spreadsheet work, same audit controls.
