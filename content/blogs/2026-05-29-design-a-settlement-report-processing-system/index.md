---
draft: true
title: Design a Settlement Report Processing System
slug: design-a-settlement-report-processing-system
date: 2026-05-29T10:00:00+0800
categories:
- Payments
tags:
- Settlement
- Reconciliation
---

## Background

Every acquirer and PSP publishes settlement reports in its own shape: schemas, delivery channels, payout cadences, and adjustment conventions. A platform routing cards, wallets, and bank transfers needs to ingest every variant and join each file to internal payment records.

The settlement report is the **source of truth** for money that actually moved. Payment records capture intent; the report shows what cleared, what was netted, and what reached the bank. Reconciliation needs both views joined reliably.

The gap is **chronic under-investment** in a settlement processing pipeline. Finance depends on it, but engineering budget and roadmap space go to payment acceptance and new methods instead. With no purpose-built replacement, teams extend a legacy pipeline through quick fixes—parser patches, manual operations, one-off scripts. Each workaround defers proper design and adds **technical debt** and **operational overhead**: brittle mappings, tacit runbook steps, on-call pages when a file format drifts, and repeated manual transforms that introduce inconsistency and toil. Those patches embed provider quirks in core systems and interleave acquirer-specific logic across shared paths, so every new onboarding carries interdependency and regression risk.

This post designs a dedicated processing pipeline to close that gap.

## Challenges

A payment platform records authorizations, captures, refunds, and chargebacks as operational events. Settlement is when funds actually move: the acquirer or PSP nets fees, reserves, and adjustments, then pays the merchant on a payout cadence. The authoritative record of that movement is the **settlement report** (also called a reconciliation file, payout report, or funding statement depending on the provider).

For each settlement window and each merchant or MID, the platform needs to answer:

- Which internal transactions settled in this payout?
- What fees, reserves, and FX adjustments explain the delta between gross and net?
- Where settlement lines, existing records, and bank statement entries disagree, and what explains exceptions?

Answering those questions at scale means clearing a stack of provider-specific obstacles in ingestion, matching, and operations:

**Heterogeneous report formats**

Each PSP and acquirer publishes its own schema: column names, date formats, amount signs, fee breakdowns, and identifier fields differ. Each payment method adds another variant. Some reports are CSV; others are fixed-width, XML, or JSON via API. A processor may expose multiple report types—transaction-level detail, summary by day, payout-level totals—that only tie together through report id or payout reference.

**Delivery channels and scheduling**

Reports arrive over SFTP, email, HTTPS API, or provider portals. Delivery time and generation schedule vary by provider, MID, and report type: some files land hours after the payout batch closes, others on T+1 or later, and LPM schemes may skip a business day entirely. Files go missing, arrive late, or arrive corrupt—each case blocking the reconciliation window finance expected.

**Report scope and organization**

Providers slice reports by master MID, sub-merchant MID, currency, or payout batch. The same merchant may receive separate files per currency or per legal entity. Correlating everything that belongs to one settlement window is non-trivial when the model is not one file per merchant per day.

**Credential and secret management**

SFTP keys, API tokens, and mailbox credentials multiply with every provider and merchant account. In legacy setups they are often embedded per integration, making rotation, access control, and environment isolation costly to change.

**Identifier alignment**

Internal systems key on orchestrator payment ids, PSP payment references, authorization codes, and network retrieval reference numbers. Reports may use any subset of those, or only provider-native ids. Overlap is incomplete, joins are ambiguous, and wrong attachments are easy to miss without explicit review.

**Versioning and amendments**

Providers rarely republish an identical file with an explicit version flag when something changes. More often, a later report for the same payout batch carries **adjustment journal lines** that correct or reverse entries from an earlier delivery. A settlement window that looked closed can gain new lines weeks later.

**File quality and partial failure**

Provider files contain malformed rows, unexpected enums, or amount fields that fail validation. Legacy pipelines often treat a file as all-or-nothing: one bad row fails the entire import and blocks reconciliation for every other line in the batch.

**Amount semantics**

Gross, net, fee, tax, and FX conversion lines use inconsistent signs and currencies across providers. Some rows carry transaction currency and settlement currency on different columns; others mix both on one amount. Fee breakdowns may appear as separate lines, embedded in net, or booked in a later report.

**Volume**

Large merchants generate millions of lines per file; volume multiplies when many providers and MIDs deliver on the same T+1 window. Pipelines sized for smaller files hit memory limits, run hours-long batch jobs, and fall behind finance close deadlines.

**Observability, monitoring, and audit**

When finance finds a mismatch, legacy pipelines often cannot trace it back to a specific file version, transformation rule, or payment id. Ingest lag, match rate, and exception volume stay invisible until someone notices a failed close or a bank deposit that does not tie out. Missing, delayed, or corrupt reports surface through manual checks rather than proactive detection. Raw provider files and normalized rows are discarded or scattered across ad hoc storage, so an audit months later cannot reconstruct what was processed or why.

## Goal

- Provide a single, generic pipeline for settlement report ingestion, normalization, and reconciliation.
- Standardize the reconciliation workflow across providers and payment methods.
- Persist settlement reports and normalized rows so history survives reprocessing and amendments.
- Expose queryable settlement data that joins reliably to payment records and bank statements.
- Make reconciliation repeatable, auditable, and automatable end-to-end.
- Onboard new providers, report types, and file formats through configuration, not large system changes.
- Let downstream consumers choose output format, column layout, grouping, and generation schedule without forking the core pipeline.

## Capabilities

- **Multi-channel ingestion**: Support SFTP, email, HTTPS API, and provider portal fetch; track expected delivery per configuration; alert on missing, delayed, or failed reports.
- **Configuration-driven formats**: Standard normalized column schema and flexible per-provider file definitions so new report types onboard without code changes.
- **Report scope and correlation**: Group and correlate files by master MID, sub-merchant MID, currency, legal entity, payout batch, or settlement window.
- **Credential management**: Centralize SFTP keys, API tokens, and mailbox credentials with rotation and per-environment isolation—not hard-coded per integration.
- **Identifier matching**: Maintain an external id map with tolerant join rules; route ambiguous matches to review instead of silent attachment.
- **Versioning and amendments**: Idempotent ingestion on report id and version; retain superseded raw files and normalized rows; re-run downstream reconciliation when a batch gains adjustment lines.
- **Partial failure handling**: Quarantine invalid rows with reason codes; continue processing the rest of the file.
- **Amount normalization**: Apply per-provider sign and currency conventions; carry transaction and settlement currency where applicable; support payment, modification, chargeback, fee, tax, FX, gross, and net line types.
- **Volume at scale**: Stream or chunk large files; commit in batch-scoped units; support idempotent replay without duplicate downstream posting.
- **Three-way reconciliation**: Match settlement lines to payment records and bank statement entries; track and explain exceptions.
- **Flexible record organization**: Store and query settlement data by merchant, MID, currency, payout window, or other configured dimensions.
- **API and metadata**: Expose normalized settlement data through an API; attach metadata for source file, provider, batch, and processing run.
- **Observability and audit**: Emit metrics on ingest lag, match rate, exception counts, and run duration; retain raw artifacts and normalized rows for the provider retention window; trace any outcome to file version, rule, and payment id.
- **Configurable output**: Let downstream consumers choose export format, column layout, grouping, and generation schedule without forking the core pipeline.

