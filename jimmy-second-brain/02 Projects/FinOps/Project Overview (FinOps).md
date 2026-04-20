---
type: problems
date: 2026-04-20
project: FinOps
author: claude
---

## Goal
Consolidate JetX/Wash24h financial data into a single source of truth — P&L visibility by station/region/company, automated reconciliation, VAT compliance, and Xero integration.

## Why
Revenue lives in the JetX op system, costs are migrating to Xero, settlements from Vietinbank + Cybersource arrive in separate files, VAT matching against EasyInvoice is manual, and the finance team assembles P&L by hand. Needed for daily ops discipline and Series A-grade financial visibility.

## Tangible Outcomes
- Daily reconciliation engine matching op transactions ↔ Vietinbank + Cybersource settlements, with tiered exceptions (unmatched OP, unmatched settlement, duplicates, amount mismatch, refunds)
- VAT compliance tracker against EasyInvoice API with per-station match rates and monthly filing exports
- Monthly P&L at station / region / company level, with machine-type and service-type revenue splits
- Automated Xero journal entry push (monthly, after finance team close approval)
- Exceptions-first finance dashboard (role-gated, CSV/Excel export)

## Open Problems
1. Define region groupings (which stations → which regions)
2. Define service-type taxonomy (wash package names/codes)
3. Vietinbank settlement file format — field mapping
4. Cybersource settlement file format — field mapping + batch-settlement explosion logic
5. EasyInvoice API — auth + endpoints
6. Xero chart-of-accounts mapping for journal entries
7. Shared cost allocation methodology (HQ overhead → stations by revenue share or flat split)
8. Dashboard tech stack decision (React vs existing cam-web pattern)
9. Historical backfill scope (24-month op data availability)
10. Reconciliation edge cases — refund matching, duplicate detection, partial/batch settlement explosion

## Source Docs
- [[brief]] — one-pager
- [[finops-brd]] — Business Requirements Doc v1.0 (Nemo, 2026-03-30)
- [[finops-deep-spec]] — deep feature spec (Nemo, 2026-03-30)

## Architecture Alignment

**Platform principles ([[jetx-platform-principles]]):**
- #2 Identity in one place — finance team auth via Keycloak JWT
- #3 Permissions as capabilities — `finance:read`, `finance:reconcile`, `finance:close`, `finance:push-xero`
- #4 Core domain in one service — FinOps FKs into platform-core for `Station`, `Customer`, `Vehicle`
- #5 Each app owns its own domain data — FinOps owns invoices, reconciliation exceptions, journal entries
- #9 Boring tech — daily batch over real-time, Postgres + dbt + FastAPI

**Data-ownership channel position ([[jetx-data-ownership-principles]]):**
- **Channel 1 (Sync API):** finance team resolving reconciliation exceptions, triggering manual Xero re-push
- **Channel 3 upstream reader:** JetX op system transactions, Vietinbank + Cybersource settlements, EasyInvoice e-invoices, Xero bills/expenses
- **Channel 3 downstream writer:** monthly journal entries pushed to Xero (Xero is the accounting system of record)

**Upstream / downstream dependencies:**
- **Identity provider:** [[Project Overview (Identity)]] — FinOps is the first real client app; its capability scopes (`finance:read`, `finance:reconcile`, `finance:close`, `finance:push-xero`) need to be registered there
- **Upstream data:** [[Project Overview (Jetx-Lakehouse)]] — Gold tables feed reconciliation, VAT, and P&L engines
- **Downstream:** Xero (accounting system of record, via journal entry push)

## FinOps as the Architecture Reference Implementation

FinOps is the first app to implement the full JetX platform architecture end-to-end — Keycloak JWT validation (via Identity), core-domain FKs into platform-core, BFF pattern for the finance dashboard, Incus + Caddy deployment. Patterns proven here get promoted into the framework docs and the new-project runbook.

## See Also
- [[Project Overview (Identity)]] — identity provider dependency; capability scopes live there
- [[Project Overview (Jetx-Lakehouse)]] — upstream data source; Gold tables feed reconciliation, VAT, and P&L engines
- [[jetx-platform-principles]] · [[jetx-data-ownership-principles]] · [[00-architecture]]
- [[Principles Found (FinOps)]] — durable principles discovered while building this project
