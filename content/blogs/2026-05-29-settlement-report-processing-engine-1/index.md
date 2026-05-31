---
draft: true
title: Settlement Report Processing Engine (1) - The Challenge
slug: settlement-report-processing-engine-1
date: 2026-05-29T10:00:00+0800
categories:
- Payments
tags:
- Settlement
- Reconciliation
---

## Background

Every acquirer and PSP publishes settlement reports in its own shape: schemas, delivery channels, payout cadences, and adjustment conventions. A platform routing cards, wallets, and bank transfers must ingest every variant and join each file to internal payment records.

The settlement report (also called a reconciliation file, payout report, or funding statement, depending on the publisher) is the **source of truth** for money that moved. Payment records capture intent; the report shows what cleared, what was netted, and what reached the bank. Reconciliation needs both views joined reliably.

Most platforms have no dedicated settlement report processing system — only a **fragmented pipeline** of ingest scripts, manual transforms, and publisher-specific workarounds. Finance depends on the pipeline, yet engineering teams **chronically under-invest** in modernizing it — roadmap capacity goes to payment acceptance and new methods. Without dedicated evolution, teams patch shared paths through manual operations and one-off scripts, and debt accumulates — brittle mappings, tacit runbooks, on-call pages when formats drift.

The following posts design a dedicated processing pipeline to close that gap.

## Challenges

A payment platform records authorizations, captures, refunds, and chargebacks as operational events. **Settlement** is when funds move: the acquirer or PSP nets fees, reserves, and adjustments, then pays the merchant on a payout cadence. The settlement report is the **authoritative record** of that movement.

For each settlement window and each merchant or MID, the platform must answer:

* Which internal transactions settled in this payout?
* What fees, reserves, and FX adjustments explain the delta between gross and net?
* Where settlement lines, payment records, and bank statement entries disagree, and what explains the exceptions?

Answering those questions at scale means clearing publisher-specific obstacles in ingestion, matching, and operations.

### Heterogeneous report formats

Each PSP and acquirer publishes its own schema: column names, date formats, amount signs, fee breakdowns, and identifier fields differ. Each payment method adds another variant. Reports arrive as CSV, fixed-width, XML, or JSON via API. A publisher may expose multiple report types—transaction-level detail, summary by day, payout-level totals—that tie together only through report id or payout reference.

### Delivery channels and scheduling

Reports arrive over SFTP, email, HTTPS API, or publisher portals. Delivery time and generation schedule vary by publisher, MID, and report type. Some files land hours after the payout batch closes; others on T+1 or later; LPM schemes may skip a business day. Files go missing, arrive late, or arrive corrupt—each case blocking the reconciliation window finance needs.

### Report scope and organization

Publishers slice reports by master MID, sub-merchant MID, currency, or payout batch. The same merchant may receive separate files per currency or per legal entity. Correlating everything that belongs to one settlement window is non-trivial when the model is not one file per merchant per day.

### Credential and secret management

SFTP keys, API tokens, and mailbox credentials multiply with every publisher and merchant account. Legacy setups often embed them per integration, which makes rotation, access control, and environment isolation costly to change.

### Identifier alignment

Internal systems key on orchestrator payment ids, PSP payment references, authorization codes, and network retrieval reference numbers. Reports may use any subset of those, or only publisher-native ids. Overlap is incomplete, joins are ambiguous, and wrong attachments are easy to miss without explicit review.

### Versioning and amendments

Publishers rarely republish an identical file with an explicit version flag when something changes. More often, a later report for the same payout batch carries **adjustment journal lines** that correct or reverse entries from an earlier delivery. A settlement window that looked closed can gain new lines weeks later.

### File quality and partial failure

Publisher files contain malformed rows, unexpected enums, or amount fields that fail validation. Legacy pipelines often treat a file as all-or-nothing: one bad row fails the entire import and blocks reconciliation for every other line in the batch.

### Amount semantics 

Gross, net, fee, tax, and FX conversion lines use inconsistent signs and currencies across publishers. Some rows carry transaction currency and settlement currency on different columns; others mix both on one amount. Fee breakdowns may appear as separate lines, embedded in net, or booked in a later report.

### Volume

Large merchants generate millions of lines per file; volume multiplies when many publishers and MIDs deliver on the same T+1 window. Pipelines sized for smaller files hit memory limits, run hours-long batch jobs, and fall behind finance close deadlines.

### Observability, monitoring, and audit

When finance finds a mismatch, legacy pipelines often cannot trace it back to a specific file version, transformation rule, or payment id. Ingest lag, match rate, and exception volume stay invisible until someone notices a failed close or a bank deposit that does not tie out. Missing, delayed, or corrupt reports surface through manual checks rather than proactive detection. Raw publisher files and normalized rows are discarded or scattered across ad hoc storage, so an audit months later cannot reconstruct what was processed or why.
