# JetX Lakehouse - Project Context

> **Document Purpose:** Capture the full context of the current data warehouse and requirements for the new lakehouse architecture.
>
> **Status:** Planning Phase - Tech Stack Discussion Pending

---

## 1. Business Context

**Company:** Wash24h - A network of 17+ automated car wash stations in Vietnam

**Business Model:**
- Pay-per-wash (QR payment, JetX Cash wallet, vouchers)
- Subscription plans (monthly/quarterly/yearly unlimited washes)
- B2B fleet services

**Key Metrics Tracked:**
| Category | Metrics |
|----------|---------|
| Revenue | Gross, net, by payment method, by station |
| Subscriptions | MRR, churn, renewals, usage |
| Customer Behavior | RFM segmentation, lifetime value |
| Station Performance | Utilization, P&L, rankings |
| Fraud Detection | Voucher abuse, plate sharing |
| Active Users | A30/A90 counts |

---

## 2. Current Architecture

```
SOURCE DATABASE (Production App - Read Only)
    │
    │  [Layer 0: Extract]
    ▼
┌─────────────────────────────────────────────┐
│  WAREHOUSE: DIMENSION & FACT TABLES         │
│  ├── dim_customers (42K rows)               │
│  ├── dim_stations (17 rows)                 │
│  ├── dim_vehicles (3.5K rows)               │
│  ├── dim_subscriptions (1.8K rows)          │
│  ├── dim_devices, dim_wash_modes, etc.      │
│  ├── fact_washes (116K rows)                │
│  ├── fact_transactions (70K rows)           │
│  └── fact_subscription_events               │
└─────────────────────────────────────────────┘
    │
    │  [Layer 1: Aggregate]
    ▼
┌─────────────────────────────────────────────┐
│  ANALYTICS TABLES                           │
│  ├── analytics_daily_orders                 │
│  ├── analytics_subscription_revenue         │
│  ├── analytics_subscription_usage           │
│  ├── analytics_customer_metrics             │
│  ├── analytics_station_performance          │
│  ├── analytics_station_pnl                  │
│  ├── analytics_fraud_indicators             │
│  └── ... (11 tables total)                  │
└─────────────────────────────────────────────┘
    │
    │  [Layer 2: Business Logic]
    ▼
┌─────────────────────────────────────────────┐
│  DATAMARTS                                  │
│  ├── datamart_active_users                  │
│  ├── datamart_revenue_dashboard             │
│  ├── datamart_station_scorecard             │
│  ├── datamart_customer_segmentation         │
│  └── datamart_fraud_report                  │
└─────────────────────────────────────────────┘
    │
    ▼
  API / WebUI / Reports
```

---

## 3. Source Database Schema

The production app uses PostgreSQL. Key tables:

### Orders & Transactions
```sql
-- orders: Every wash order
- id, order_number, station_id, customer_id
- sub_total, discount_amount, total_amount
- payment_method, payment_gateway
- status, created_at, completed_at
- order_items (JSONB): wash details, pricing

-- transactions: Payment records
- id, order_id, customer_id
- amount, payment_method, gateway_response
- status, created_at
```

### Customers & Vehicles
```sql
-- users: Customer accounts
- id, name, email, phone
- wallet_balance (JetX Cash)
- created_at, last_login

-- vehicles: Registered vehicles
- id, license_plate, customer_id
- brand, model, color
- created_at
```

### Subscriptions
```sql
-- subscriptions: Active subscriptions
- id, customer_id, plan_id
- status (active/expired/cancelled)
- current_period_start_date, current_period_end_date
- wash_count_this_period, wash_limit

-- membership_plans: Plan definitions
- id, name, price, duration_days
- wash_limit, description
```

### Stations & Devices
```sql
-- stations: Wash locations
- id, name, address, lat, lng
- status, created_at

-- devices: Wash machines
- id, station_id, device_number
- status, last_heartbeat
```

---

## 4. Current Warehouse Tables

### Dimension Tables (dim_*)

**dim_customers**
```sql
- customer_id (PK), customer_name, email, phone
- first_order_date, created_at
- is_verified, verification_date
```

**dim_stations**
```sql
- station_id (PK), station_name
- address, latitude, longitude
- tier (A/B/C), region
- opened_date, status
```

**dim_subscriptions**
```sql
- subscription_id (PK), customer_id
- plan_id, plan_name, plan_price
- status, start_date, end_date
- wash_limit, wash_count_this_period
```

**dim_vehicles**
```sql
- vehicle_id (PK), license_plate
- customer_id, brand, model, color
- first_seen_date, last_seen_date
- owner_count (fraud indicator)
```

### Fact Tables (fact_*)

**fact_washes** (grain: one row per wash)
```sql
- wash_id (PK), order_id, customer_id, station_id
- vehicle_id, device_id, wash_mode_id
- wash_date, start_time, end_time, duration_seconds
- is_subscription_wash, subscription_id
- gross_amount, discount_amount, net_amount
- payment_method
- latitude, longitude (GPS)
- alarm_codes (JSONB)
```

**fact_transactions** (grain: one row per payment)
```sql
- transaction_id (PK), order_id, customer_id
- transaction_date, amount
- payment_method, payment_gateway
- status (success/failed/pending)
- gateway_response (JSONB)
```

**fact_subscription_events** (grain: one row per lifecycle event)
```sql
- event_id (PK), subscription_id, customer_id
- event_date, event_type (new/renewed/soft_churned/churned/reactivated)
- plan_id, mrr_impact
- days_since_expiry (for churn events)
```

### Analytics Tables (analytics_*)

**analytics_daily_orders**
```sql
- date, station_name, payment_method
- order_count, unique_customers
- total_revenue, net_revenue, avg_order_value
- discount_amount, tax_amount
```

**analytics_subscription_revenue**
```sql
- date, station_name
- active_subscriptions, mrr
- new_subscriptions, churned_subscriptions
- soft_churned_subscriptions, reactivated_subscriptions
```

**analytics_subscription_usage**
```sql
- date, subscription_id, customer_id
- plan_name, wash_count, wash_limit
- usage_rate, washes_per_month (normalized)
- is_idle (no usage in 7+ days)
- days_remaining
```

**analytics_customer_metrics**
```sql
- customer_id, customer_name, phone
- total_orders, total_lifetime_value
- first_order_date, last_order_date
- recency_days, frequency_count, monetary_total
- rfm_segment (Champions, Loyal, At Risk, etc.)
- favorite_station, favorite_payment_method
```

**analytics_station_performance**
```sql
- date, station_id, station_name
- order_count, unique_customers
- total_revenue, net_revenue
- new_customers, returning_customers
- avg_order_value
```

**analytics_station_pnl**
```sql
- month, station_name
- total_revenue, total_cost
- gross_profit, gross_margin_pct
- (joins with station_monthly_costs config table)
```

### Datamart Tables (datamart_*)

**datamart_active_users**
```sql
- snapshot_date, station_id, station_name
- a30_count (active last 30 days)
- a90_count (active last 90 days)
```

**datamart_revenue_dashboard**
```sql
- date, period_type (daily/weekly/monthly)
- total_revenue, net_revenue, order_count
- subscription_revenue, topup_revenue
- active_subscriptions, mrr
- growth metrics vs previous period
```

**datamart_station_scorecard**
```sql
- date, station_id, station_name
- revenue_score, growth_score, retention_score
- overall_score, rank
- performance_tier (Excellent/Good/Fair/Poor)
```

**datamart_customer_segmentation**
```sql
- customer_id, segment_date
- rfm_segment, value_tier, lifecycle_stage
- recency_score, frequency_score, monetary_score
- is_vip, is_at_risk, is_churned
- recommended_action
```

---

## 5. ETL Pipeline

### Job Organization (19 jobs total)

**Layer 0 (Source → DW):** Requires SOURCE database access
- 8 dimension table jobs (run once, no date params)
- 2 fact table jobs (date range params)

**Layer 1 (DW → Analytics):** DW only, no SOURCE needed
- 11 analytics jobs (most take date range)

**Layer 2 (Analytics → Datamarts):** DW only
- 6 datamart jobs

### Scheduling
- **Hourly at :05** - Today's data, all layers
- **Daily at 2AM** - Backfill last 2 days

### Current Tech Stack
| Component | Technology |
|-----------|------------|
| Database | PostgreSQL (self-hosted on DigitalOcean Droplet) |
| ETL | Python + Pandas |
| ORM | SQLAlchemy |
| Migrations | Alembic |
| Deployment | Docker |

---

## 6. Business Logic Definitions

### Churn Lifecycle
```
Active → Soft-Churn → Reactivated (success)
                   → True Churn (90+ days expired)

Definitions:
- Soft-Churn: Subscription expired, customer may renew
- Reactivated: Was soft-churned, renewed within 90 days
- True Churn: 90+ days since expiration (lost customer)
```

### RFM Segmentation
```
Scores (1-5 scale):
- Recency: Days since last order
- Frequency: Order count in period
- Monetary: Total spend

Segments:
- Champions (R=5, F=5, M=5)
- Loyal Customers (R=4-5, F=4-5)
- Potential Loyalists (R=4-5, F=1-3)
- At Risk (R=1-2, F=4-5)
- Can't Lose Them (R=1-2, F=4-5, M=5)
- Hibernating (R=1-2, F=1-2)
- Lost (R=1, F=1)
```

### Active Users
```
A30: Unique customers with at least 1 wash in last 30 days
A90: Unique customers with at least 1 wash in last 90 days
```

### Normalized Usage
```
washes_per_month = total_washes_this_period / days_in_period * 30
(Normalizes monthly/quarterly/yearly billing cycles)
```

---

## 7. Current System Assessment

### What Works Well
- [x] 3-layer architecture (fact/analytics/datamart) works well
- [x] Separation of SOURCE access (Layer 0) from DW-only jobs
- [x] Incremental date-based ETL for most jobs
- [x] Churn tracking with soft-churn/reactivation states
- [x] RFM segmentation is production-ready
- [x] Station P&L with configurable costs
- [x] All 19 ETL jobs stable and running on schedule

### What Needs Improvement

**Architecture:**
- [ ] No data versioning or time-travel
- [ ] No schema evolution tracking
- [ ] No data quality checks/validation
- [ ] No lineage tracking
- [ ] Single PostgreSQL instance (no separation of storage/compute)

**ETL:**
- [ ] Python + Pandas doesn't scale well for large datasets
- [ ] No incremental/CDC for fact tables (full refresh each run)
- [ ] No parallelization of ETL jobs
- [ ] No retry/recovery mechanisms
- [ ] No data contracts or schema validation

**Analytics:**
- [ ] No real-time/streaming data
- [ ] No ML feature store
- [ ] Limited historical snapshots (SCD Type 1 only)
- [ ] No data catalog or discovery

---

## 8. Lakehouse Vision

### Goals

1. **Separate storage and compute**
   - Object storage (S3/MinIO) for data lake
   - Query engine (DuckDB/Trino/Spark) for compute

2. **Modern table formats**
   - Delta Lake or Apache Iceberg
   - Time-travel, schema evolution, ACID transactions

3. **Data quality**
   - Data contracts
   - Schema validation
   - Quality checks (Great Expectations or similar)

4. **Incremental processing**
   - CDC from source
   - Incremental materialization
   - Streaming ingestion option

5. **Data governance**
   - Data catalog
   - Lineage tracking
   - Access control

6. **Cost-effective**
   - Can run on single node for small scale
   - Can scale horizontally when needed
   - Uses open formats (no vendor lock-in)

---

## 9. Requirements

### Must Have
- Reproduce all current analytics (19 ETL jobs worth)
- Support same business logic (churn, RFM, P&L)
- API-compatible with current system (can swap in)
- Run on modest infrastructure (single Droplet to start)

### Nice to Have
- Real-time dashboard updates
- ML feature store
- Self-service analytics (SQL interface)
- Data quality monitoring

### Tech Preferences
- Python-based (team knows Python)
- Open source preferred
- Can run locally for development
- Docker-based deployment

---

## 10. Data Scale Reference

| Table | Row Count | Growth Rate |
|-------|-----------|-------------|
| dim_customers | 42K | ~500/month |
| dim_stations | 17 | Slow |
| dim_vehicles | 3.5K | ~200/month |
| dim_subscriptions | 1.8K | ~100/month |
| fact_washes | 116K | ~5K/month |
| fact_transactions | 70K | ~3K/month |

**Current Total Data Size:** ~500MB (PostgreSQL)

---

## 11. Open Questions for Tech Stack Discussion

1. What lakehouse architecture would you recommend for this scale?
2. Should we use Delta Lake, Iceberg, or Hudi?
3. What orchestration tool (Airflow, Dagster, Prefect)?
4. How should we handle the transition from current system?
5. What's the minimum viable lakehouse we can start with?

---

## Appendix: Table Relationships

```
                    ┌──────────────┐
                    │ dim_stations │
                    └──────┬───────┘
                           │
    ┌──────────────┐       │       ┌─────────────────┐
    │dim_customers │───────┼───────│dim_subscriptions│
    └──────┬───────┘       │       └────────┬────────┘
           │               │                │
           │    ┌──────────┴──────────┐     │
           └────┤    fact_washes      ├─────┘
                └──────────┬──────────┘
                           │
                ┌──────────┴──────────┐
                │  fact_transactions  │
                └─────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  fact_subscription_    │
              │       events           │
              └────────────────────────┘
```
