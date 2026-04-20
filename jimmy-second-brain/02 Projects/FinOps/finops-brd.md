# Business Requirements Document (BRD)
# FinOps Platform — JetX / Wash24h

_Version: 1.0_
_Date: 2026-03-30_
_Author: Nemo_
_Status: Draft — pending Jimmy review_

---

## 1. Executive Summary

JetX/Wash24h operates 17 car wash stations with 21 A5 vending machines across multiple regions. The business currently lacks consolidated financial visibility — revenue, costs, VAT compliance, and P&L are managed in disparate systems.

The FinOps Platform will consolidate all financial data into a single source of truth, delivering P&L visibility down to station, region, and company level, with automated reconciliation, VAT compliance tracking, and Xero integration.

---

## 2. Business Context

### 2.1 Company Structure
- **Legal entity:** Wash24h (operating company)
- **Brand:** JetX
- **Scale:** 17 stations, 21 A5 vending machines, growing

### 2.2 Current State (Pain Points)
- Revenue data lives in the JetX operational system (car wash transactions)
- Costs currently in accounting software, migrating to Xero
- No automated reconciliation between op revenue and bank settlements
- VAT compliance tracking is manual — e-invoices not systematically matched
- No unified P&L view — finance team assembles reports manually
- Two payment processors (Vietinbank, Cybersource/Visa) with separate settlement files

### 2.3 Target State
- Single FinOps platform as **source of truth** for all financial data
- Daily automated reconciliation (op transactions ↔ bank settlements)
- Real-time VAT compliance status per station
- Monthly P&L by station / region / company
- Automated journal entry push to Xero (Xero = downstream reporting, not source of truth)

---

## 3. Stakeholders

| Role | Party | Involvement |
|------|-------|-------------|
| Product Owner | Jimmy | Approves requirements and design |
| Finance Team | Wash24h accounting | Primary end users |
| Engineering | JetX tech team | Build and maintain |
| External: Accounting | Xero | Downstream journal entry recipient |
| External: E-invoice | EasyInvoice | VAT invoice API source |
| External: Payment | Vietinbank | Bank settlement feed |
| External: Payment | Cybersource (Visa) | Card payment settlement feed |

---

## 4. Scope

### 4.1 In Scope
- Revenue ingestion from JetX operational system
- Payment reconciliation (Vietinbank + Cybersource settlements)
- VAT compliance tracking via EasyInvoice API
- Cost ingestion from Xero (bills/expenses)
- P&L engine — station / region / company / machine type / service type
- Xero journal entry push (daily batch)
- Finance dashboard (exceptions-first UI)

### 4.2 Out of Scope (v1)
- Walk-in vs subscription P&L breakdown (too complex to reliably attribute)
- Real-time processing (daily batch is sufficient)
- Payroll integration
- Multi-currency (VND only)
- Mobile app

---

## 5. Functional Requirements

### 5.1 Revenue Ingestion

| ID | Requirement |
|----|-------------|
| REV-01 | Ingest all car wash transactions from JetX op system daily (Bronze → Silver) |
| REV-02 | Ingest A5 vending machine transactions daily |
| REV-03 | Tag each transaction with: station_id, region_id, machine_type, service_type |
| REV-04 | Support historical backfill for past 24 months |

### 5.2 Payment Reconciliation

| ID | Requirement |
|----|-------------|
| REC-01 | Ingest daily settlement files from Vietinbank |
| REC-02 | Ingest daily settlement files from Cybersource (Visa) |
| REC-03 | Match op transactions to bank settlements by: amount, date range (T+0 to T+2), station |
| REC-04 | Flag unmatched transactions (op side) — revenue not settled |
| REC-05 | Flag unmatched settlements (bank side) — settlement without op record |
| REC-06 | Flag duplicate settlements |
| REC-07 | Finance dashboard shows reconciliation status: green (matched) / red (unmatched) per station per day |
| REC-08 | Allow finance team to manually resolve exceptions with notes |

### 5.3 VAT Compliance

| ID | Requirement |
|----|-------------|
| VAT-01 | Pull issued e-invoices daily from EasyInvoice API |
| VAT-02 | Match e-invoices to op transactions (1:1 by transaction reference or amount+date) |
| VAT-03 | Calculate match rate per station per day |
| VAT-04 | Flag transactions missing e-invoice |
| VAT-05 | Flag e-invoices with invalid/missing data |
| VAT-06 | Finance dashboard shows VAT compliance status per station |
| VAT-07 | Export VAT compliance report (monthly, for tax filing) |

### 5.4 Cost Ingestion

| ID | Requirement |
|----|-------------|
| COST-01 | Pull bills/expenses from Xero API daily |
| COST-02 | Tag costs with: station_id (where applicable), cost_category, cost_type |
| COST-03 | Support shared costs (HQ-level) — allocate to stations by revenue share or flat split |
| COST-04 | Support manual cost upload (CSV) for costs not in Xero |

### 5.5 P&L Engine

| ID | Requirement |
|----|-------------|
| PNL-01 | Calculate monthly P&L: Revenue − Costs = Gross Profit per station |
| PNL-02 | Roll up P&L to region level (sum of stations in region) |
| PNL-03 | Roll up P&L to company level (Wash24h total) |
| PNL-04 | Break down revenue by machine type (car wash vs A5) |
| PNL-05 | Break down revenue by service type (wash packages) |
| PNL-06 | Daily visibility (cumulative MTD), monthly close for actuals |
| PNL-07 | Compare actuals vs prior month / prior year |

### 5.6 Xero Integration

| ID | Requirement |
|----|-------------|
| XERO-01 | Push monthly P&L summary as journal entries to Xero |
| XERO-02 | Journal entries tagged by station / region / cost category |
| XERO-03 | One push per month after finance team approves close |
| XERO-04 | Log all push attempts with status (success/fail) |
| XERO-05 | Finance team can trigger manual re-push if needed |

### 5.7 Finance Dashboard

| ID | Requirement |
|----|-------------|
| DASH-01 | Reconciliation exceptions view — unmatched transactions by station/date |
| DASH-02 | VAT compliance exceptions view — missing invoices by station/date |
| DASH-03 | P&L view — station / region / company, selectable date range |
| DASH-04 | Revenue breakdown — by machine type and service type |
| DASH-05 | Export to CSV/Excel for all views |
| DASH-06 | Role-based access (finance team only) |

---

## 6. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| Performance | Dashboard loads in < 3s for any date range |
| Availability | 99.5% uptime (batch jobs can fail and retry; dashboard is ops-critical) |
| Data freshness | All data updated by 8:00 AM daily |
| Data retention | 3 years minimum |
| Security | Finance data access restricted to authorized users; no PII in data warehouse |
| Auditability | All data ingestion jobs logged with timestamps, record counts, error rates |

---

## 7. Technical Architecture

### 7.1 Data Flow

```
Sources
  ├── JetX Op System (transactions, service types)    → Bronze (raw)
  ├── Vietinbank (settlement files)                   → Bronze (raw)
  ├── Cybersource/Visa (settlement files)             → Bronze (raw)
  ├── EasyInvoice API (e-invoices)                    → Bronze (raw)
  └── Xero API (bills/expenses)                       → Bronze (raw)

          ↓ dbt models

Silver (Iceberg) — cleaned, normalized
  ├── transactions
  ├── payment_settlements
  ├── invoices
  └── costs

          ↓ dbt models

Gold (DuckDB) — business logic
  ├── revenue_reconciliation_daily
  ├── vat_compliance_status
  ├── cost_by_category_station
  ├── pnl_station_monthly
  ├── pnl_region_monthly
  └── pnl_company_monthly

          ↓

FastAPI (/finance/* endpoints) → Finance Dashboard
                               → Xero push job
```

### 7.2 Stack
- **Storage:** JetX Lakehouse (Iceberg + DuckDB)
- **Transformation:** dbt (SQL-first, Gold layer materialization)
- **API:** FastAPI (`/finance` endpoints)
- **Scheduler:** Existing cron / Airflow (TBD)
- **E-invoice:** EasyInvoice API
- **Accounting:** Xero API
- **Payment:** Vietinbank settlement files + Cybersource settlement files

### 7.3 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| FinOps is source of truth; Xero is downstream | Xero is the accounting system of record, but financial data originates in JetX op systems — FinOps bridges them |
| Daily batch (not real-time) | Finance team works on daily/monthly cadence; real-time adds complexity without value |
| Exceptions-first dashboard | Finance team doesn't need to see all 17 stations every day — only problems |
| EasyInvoice API for VAT | EasyInvoice has API access; no manual export needed |

---

## 8. P&L Dimensions

| Dimension | Values | Notes |
|-----------|--------|-------|
| Station | 17 stations | Primary granularity |
| Region | TBD groupings | Roll-up of stations |
| Company | Wash24h | Single legal entity (v1) |
| Machine type | Car wash, A5 vending | Revenue split only (costs shared) |
| Service type | Wash packages (types TBD) | Revenue split only |

---

## 9. Open Items (Post-BRD)

| # | Item | Owner |
|---|------|-------|
| 1 | Define region groupings (which stations → which regions) | Jimmy |
| 2 | Define service type taxonomy (wash package names/codes) | Jimmy |
| 3 | Vietinbank settlement file format — field mapping | Finance team |
| 4 | Cybersource settlement file format — field mapping | Finance team |
| 5 | EasyInvoice API documentation — auth + endpoints | Engineering |
| 6 | Xero journal entry format — chart of accounts mapping | Finance team |
| 7 | Shared cost allocation methodology | Finance team |
| 8 | Dashboard tech stack (React? existing cam-web pattern?) | Jimmy |
| 9 | Historical backfill scope (24 months of op data available?) | Engineering |

---

## 10. Milestones (Proposed)

| Phase | Deliverable | Estimate |
|-------|-------------|----------|
| Design | Answer open items, finalize data model, API contracts | 2 weeks |
| Phase 1 | Revenue ingestion + reconciliation engine + dashboard | 4 weeks |
| Phase 2 | VAT compliance layer + EasyInvoice integration | 2 weeks |
| Phase 3 | Cost ingestion (Xero) + P&L engine | 3 weeks |
| Phase 4 | Xero journal entry push | 1 week |
| UAT | Finance team testing + sign-off | 2 weeks |

_Total estimate: ~14 weeks from design-complete_

---

## 11. Approval

| Approver | Role | Status |
|----------|------|--------|
| Jimmy | Product Owner | ⏳ Pending |
| Finance Lead | End User | ⏳ Pending |

---

_BRD v1.0 — Nemo — 2026-03-30_
