# FinOps Platform — Deep Feature Spec
> /superpowers thinking | Author: Nemo | Date: 2026-03-30

---

## 1. Revenue Reconciliation Engine

### What it needs to do
Match every car wash / A5 transaction in the JetX op system against settlements arriving from Vietinbank and Cybersource. Daily batch. Flag anything that doesn't match.

### The matching problem
This is harder than it looks. Payment flows:
1. Customer pays → op system records transaction T (amount, station, timestamp, payment_ref)
2. Bank settles → settlement file arrives T+0 to T+2 (bank_ref, amount, settlement_date)
3. We match T.payment_ref ↔ settlement.bank_ref

Problems to solve:
- **Timing gaps:** Customer pays Monday, bank settles Wednesday. Matching window needs to be 72h, not same-day.
- **Partial batching:** Cybersource batches multiple transactions into one settlement record (batch settlement). Need to explode batches before matching.
- **Currency rounding:** VND has no decimals. Should be fine, but validate.
- **Refunds:** Refund settlement comes in as negative amount. Need separate refund matching logic.
- **Duplicate detection:** Same bank_ref appearing twice = duplicate settlement (over-credited). Flag immediately.
- **Multi-gateway splits:** Some transactions may route through multiple processors depending on payment method.

### Matching algorithm
```
For each unmatched op transaction T:
  1. Look for settlement S where:
     - S.amount == T.amount (exact match)
     - S.settlement_date BETWEEN T.transaction_date AND T.transaction_date + 3 days
     - S.station_id == T.station_id (if available in settlement file)
     - S.payment_ref LIKE T.payment_ref (fuzzy if bank truncates)
  2. If found: mark T as MATCHED, link T.settlement_id = S.id
  3. If not found after 3 days: mark T as UNMATCHED_OP (revenue not settled)
  4. Any S without a T after 1 day: mark S as UNMATCHED_SETTLEMENT (settlement without transaction)
```

### Exception tiers
| Code | Meaning | Urgency | Auto-resolve? |
|------|---------|---------|---------------|
| UNMATCHED_OP | Transaction in op system, no settlement | Medium (expected < T+3) | Yes — auto-clears if late settlement arrives |
| UNMATCHED_SETTLEMENT | Settlement with no op transaction | HIGH | No — manual review |
| DUPLICATE_SETTLEMENT | Same bank_ref settled twice | CRITICAL | No — immediate alert |
| AMOUNT_MISMATCH | Refs match but amounts differ | HIGH | No — manual review |
| REFUND_UNMATCHED | Refund settlement with no linked original | Medium | No — manual review |

### Dashboard view
```
Reconciliation — March 2026

Station         | Matched | Unmatched OP | Unmatched Settlement | Duplicates
FPT-HCM         | 98.2%   | 12           | 0                    | 0
Thu Duc         | 100%    | 0            | 0                    | 0
Binh Thanh      | 96.1%   | 31           | 2                    | 0
...

Drill down: click station → day view → transaction list
```

### Data model additions needed
```sql
ALTER TABLE transactions ADD COLUMN settlement_id UUID REFERENCES settlements(id);
ALTER TABLE transactions ADD COLUMN reconciliation_status TEXT; -- MATCHED, UNMATCHED_OP, etc.
ALTER TABLE transactions ADD COLUMN reconciliation_resolved_at TIMESTAMPTZ;
ALTER TABLE transactions ADD COLUMN reconciliation_note TEXT;

CREATE TABLE settlements (
  id UUID PRIMARY KEY,
  bank TEXT NOT NULL, -- vietinbank | cybersource
  bank_ref TEXT NOT NULL UNIQUE,
  amount BIGINT NOT NULL, -- VND, no decimals
  settlement_date DATE NOT NULL,
  station_id UUID REFERENCES stations(id),
  raw_data JSONB, -- original settlement file row
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE reconciliation_exceptions (
  id UUID PRIMARY KEY,
  exception_code TEXT NOT NULL,
  transaction_id UUID REFERENCES transactions(id),
  settlement_id UUID REFERENCES settlements(id),
  detected_at TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  resolved_by UUID REFERENCES users(id),
  resolution_note TEXT
);
```

---

## 2. VAT Compliance Engine

### The Vietnamese VAT context
Every taxable transaction must have a corresponding e-invoice (hóa đơn điện tử) issued within the same day (or by the end of the billing period for subscription). EasyInvoice is the provider.

Missing invoices = VAT compliance risk = tax authority penalties.

### What we need to track
1. Was an e-invoice issued for this transaction? (1:1 match)
2. Is the invoice valid? (correct amount, correct tax code, not cancelled)
3. What's the match rate per station per day?

### EasyInvoice API integration
```
EasyInvoice API calls needed:
GET /invoices?date={date}&station={station_tax_code}
  → returns list of issued invoices for that day

GET /invoices/{invoice_id}
  → returns full invoice details

POST /invoices  (future — for auto-issue if needed)
```

Fields to map:
- `invoice.amount` ↔ `transaction.amount`
- `invoice.issued_at` ↔ `transaction.created_at` (same day tolerance)
- `invoice.buyer_tax_code` — may be empty for retail customers (acceptable)
- `invoice.status` — ISSUED | CANCELLED | REPLACED

### Matching logic
```
For each transaction T on date D:
  Look for invoice I where:
    - I.amount == T.amount
    - I.issued_at DATE == T.created_at DATE
    - I.station_tax_code == T.station.tax_code
    - I.status == ISSUED

  If found: mark T as VAT_COMPLIANT
  If not found by end of D+1: mark T as VAT_MISSING
  If invoice found but cancelled: VAT_INVALID
```

### Compliance tiers
| Status | Meaning | Action |
|--------|---------|--------|
| VAT_COMPLIANT | Invoice issued and valid | None |
| VAT_MISSING | No invoice found | Finance must chase or re-issue |
| VAT_INVALID | Invoice cancelled/replaced | Finance must verify replacement |
| VAT_LATE | Invoice issued but next day | Log for audit trail |

### Dashboard view
```
VAT Compliance — March 2026

Station         | Transactions | Invoiced | Missing | Rate
FPT-HCM         | 1,240        | 1,238    | 2       | 99.8%
Thu Duc         | 890          | 845      | 45      | 94.9%
...

Monthly export for tax filing: [Download CSV]
```

---

## 3. Cost Ingestion & Allocation

### The allocation problem
Costs split into two types:
1. **Direct costs** — clearly belong to one station (electricity at FPT-HCM, staff at Thu Duc)
2. **Shared costs** — belong to the company/HQ (tech team salaries, office rent, insurance, bank fees)

For P&L to make sense at station level, shared costs must be allocated down.

### Allocation methods (pick per cost category)
| Method | When to use | Formula |
|--------|------------|---------|
| Revenue share | Operating costs tied to revenue generation | Station revenue / Total revenue × Shared cost |
| Equal split | Costs that benefit all stations equally | Shared cost / Number of active stations |
| Headcount | Staff-related shared costs | Station headcount / Total headcount × Cost |
| Direct only | Don't allocate — keep at HQ level | Show as "Unallocated" in station P&L |

### Cost categories
```
Direct costs:
  - Electricity (per station)
  - Water (per station)
  - Cleaning chemicals (per station)
  - Equipment maintenance (per station)
  - Lease/rent (per station)
  - Local staff wages (per station)

Shared costs (allocate by revenue share):
  - Tech team salaries
  - Cloud infrastructure (DO, etc.)
  - Payment processing fees (Vietinbank, Cybersource)
  - Insurance
  - Accounting/legal fees

Shared costs (don't allocate — HQ level only):
  - Executive salaries
  - Office rent
  - M&A / strategic costs
```

### Xero pull integration
```
Xero API calls:
GET /api.xro/2.0/Accounts          → chart of accounts
GET /api.xro/2.0/Invoices?Type=ACCPAY&DateFrom={date}  → bills (accounts payable)
GET /api.xro/2.0/BankTransactions  → payments made

Map Xero account codes → FinOps cost categories:
  Xero 6100 (Electricity) → ELECTRICITY
  Xero 7200 (Wages) → WAGES_DIRECT or WAGES_SHARED
  etc. (mapping table needed from finance team)
```

---

## 4. P&L Engine

### Output model
Three Gold tables, all monthly grain:

**pnl_station_monthly**
```
station_id | year | month |
revenue_total | revenue_carwash | revenue_a5 |
revenue_by_service_type (jsonb) |
costs_direct | costs_allocated | costs_total |
gross_profit | gross_margin_pct |
transaction_count | avg_revenue_per_transaction
```

**pnl_region_monthly**
```
region_id | year | month |
revenue_total | costs_total | gross_profit | gross_margin_pct |
station_count | top_station_id | bottom_station_id
```

**pnl_company_monthly**
```
company_id | year | month |
revenue_total | costs_total | gross_profit | gross_margin_pct |
ebitda (if depreciation tracked) |
yoy_revenue_growth | mom_revenue_growth
```

### dbt model dependency graph
```
Silver:
  transactions → stg_transactions
  settlements  → stg_settlements
  invoices     → stg_invoices
  xero_bills   → stg_costs

Gold:
  stg_transactions + stg_settlements → revenue_reconciliation_daily
  stg_transactions + stg_invoices    → vat_compliance_status
  stg_costs → cost_by_category_station (with allocation applied)

  revenue_reconciliation_daily + cost_by_category_station
    → pnl_station_monthly
    → pnl_region_monthly
    → pnl_company_monthly
```

### Key metrics to surface
- **Gross margin %** per station (healthy target TBD — finance team to define)
- **Revenue per wash** — trending up/down?
- **Cost per wash** — direct + allocated
- **EBITDA** — if depreciation data available from Xero
- **Revenue MTD vs last month** — is this month pacing ahead/behind?
- **Top 3 / Bottom 3 stations** by margin — quick health check

---

## 5. Xero Journal Entry Push

### When to push
- Once per month, after finance team approves close
- Finance team clicks "Push to Xero" in dashboard → confirms → push happens
- System logs the push, shows success/fail per journal entry

### What to push
One journal entry per station per month:
```
DR: Revenue (Xero account: 2000-series)     → station revenue
CR: Accounts Receivable clearing account    → offset

DR: Cost of Sales (Xero account: 5000-series) → direct costs
DR: Overhead Allocation (6000-series)          → allocated shared costs
CR: Accounts Payable clearing account          → offset
```

### Xero API calls needed
```
POST /api.xro/2.0/ManualJournals
{
  "Narration": "FinOps monthly close - FPT-HCM - March 2026",
  "JournalLines": [
    { "AccountCode": "2000", "Description": "Revenue", "LineAmount": 125000000 },
    { "AccountCode": "9999", "Description": "Clearing", "LineAmount": -125000000 }
  ]
}
```

### Idempotency
- Each push tagged with `finops_ref: {station_id}_{year}_{month}`
- Before pushing, check if journal already exists with that ref
- If exists and approved → skip (no duplicate)
- If exists and voided → re-push allowed
- Re-push always requires finance team re-approval

---

## 6. Finance Dashboard — UX Detail

### Information architecture
```
Dashboard
├── Overview (company-level KPIs, current month MTD)
├── Reconciliation
│   ├── Summary (match rates by station)
│   └── Exceptions (unmatched list, filterable)
├── VAT Compliance
│   ├── Summary (compliance rates by station)
│   └── Exceptions (missing invoices list)
├── P&L
│   ├── Company view
│   ├── Region view
│   └── Station view (drill-down)
├── Costs
│   └── Cost breakdown by category + allocation view
└── Xero Sync
    └── Push history + manual trigger
```

### Exception workflow
Finance team shouldn't need to look at green items — exceptions only.
```
Reconciliation Exceptions:
[ Filter: Station ] [ Date range ] [ Type ] [ Status: Open ]

| Date       | Station    | Amount    | Type              | Age  | Action      |
| 2026-03-28 | Thu Duc    | 450,000d  | UNMATCHED_OP      | 2d   | [Resolve]   |
| 2026-03-27 | Binh Thanh | 1,200,000d| UNMATCHED_SETTLE  | 3d   | [Investigate]|

Click [Resolve] → notes field → mark resolved
```

### Export requirements
Every table/view needs CSV export. Finance uses Excel.
- Reconciliation exceptions → for bank dispute follow-up
- VAT compliance → for tax authority submissions
- P&L → for management reporting
- All exports: UTF-8 with BOM (Vietnamese Excel needs this)

---

## 7. Data Pipeline Architecture

### Daily job schedule (all UTC+7)
```
02:00 — Ingest Vietinbank settlement files (manual upload or SFTP pull)
02:15 — Ingest Cybersource settlement files
02:30 — Pull EasyInvoice API (yesterday's invoices)
02:45 — Pull Xero API (yesterday's bills/payments)
03:00 — dbt run (Silver models)
04:00 — dbt run (Gold models — reconciliation, VAT, costs)
05:00 — dbt run (P&L models)
06:00 — Dashboard data refresh
07:30 — Alert job: email/Lark to finance team if exceptions > threshold
08:00 — Finance team starts work; all data ready
```

### Error handling
- Each job retries 3x with exponential backoff
- If job fails after retries → alert to Nemo + finance team
- Dashboard shows data freshness timestamp — finance team can see if data is stale
- Failed jobs don't block downstream unless critical dependency

### Settlement file ingestion
Vietinbank and Cybersource likely deliver settlement files via:
- SFTP pickup (scheduled pull)
- Email attachment (manual download → upload to system)
- Direct API (if available)

**Assumption:** Manual upload to start. Auto-SFTP in v2.

---

## 8. What's Still Unknown (Engineering Blockers)

These need answers before we can finalize the data model and start building:

| # | Unknown | Why it blocks us |
|---|---------|-----------------|
| 1 | Region definitions | Can't build region rollup without station→region mapping |
| 2 | Service type codes | Revenue breakdown by service type needs the taxonomy |
| 3 | Vietinbank settlement format | Field names, batch structure, date formats |
| 4 | Cybersource settlement format | Same — can't write the ingestion parser |
| 5 | EasyInvoice API auth + rate limits | Can't build the connector |
| 6 | Xero chart of accounts | Can't map costs or structure journal entries |
| 7 | Cost allocation methodology decision | Finance team must decide revenue-share vs equal-split per category |
| 8 | A5 transaction data structure | Does A5 feed into the same op system? Separate? |
| 9 | Historical backfill availability | 24 months of data — is it in the current Silver tables? |

---

_/superpowers spec — Nemo — 2026-03-30_
_Ready to feed into engineering planning once open items resolved._
