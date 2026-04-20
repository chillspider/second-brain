# JetX Lakehouse - Complete Gold Layer Specification

**Date**: 2026-01-29
**Status**: Planning Complete - Ready for Implementation
**Coverage**: 11 of 12 Analytics App Features (P&L deferred for FinOps)

---

## Executive Summary

The gold layer consists of **13 business-ready analytical tables** designed to power:
- Analytics dashboards (Metabase, PowerBI)
- Composable CRM for B2C customers
- API endpoints (datamart API)
- ML feature store
- Executive reporting

**Architecture Principle**: Gold tables are the **single source of truth** for all downstream systems. Analytics apps and CRM query these tables directly via SQL.

---

## Gold Layer Tables Overview

| # | Table Name | Grain | Status | Week | Purpose |
|---|------------|-------|--------|------|---------|
| 1 | gold_daily_revenue | Daily per station | ✅ Exists | - | Revenue & cash flow metrics |
| 2 | gold_station_performance | Daily per station | ✅ Exists | - | Station ops & active users |
| 3 | gold_subscription_mrr | Monthly per station | ⚠️ Fix needed | W1 | MRR tracking & churn |
| 4 | gold_customer_segments | Per customer | 📝 Ready | W1 | RFM segmentation |
| 5 | gold_hourly_usage | Hourly per station/DOW | 📝 Ready | W1 | Peak hours & capacity |
| 6 | gold_live_snapshot | Per station (current) | 🆕 Build | W1 | Real-time dashboard |
| 7 | gold_kpi_summary | Network/monthly/station | 🆕 Build | W1 | Executive KPIs |
| 8 | gold_vehicle_analytics | Per vehicle | 🆕 Build | W1 | Vehicle tracking |
| 9 | gold_station_summary | Per station (all-time) | 🆕 Build | W1 | Station totals |
| 10 | gold_wash_mode_performance | Monthly per station/mode | 🆕 Build | W1 | Product mix |
| 11 | gold_customer_behavior | Per customer | 🆕 Build | W2 | Customer 360 (CRM) |
| 12 | gold_cohort_retention | Cohort month × retention month | 🆕 Build | W2 | Retention analysis |
| 13 | gold_wallet_economy | Per customer + monthly trends | 🆕 Build | W2 | Wallet analytics |
| - | gold_voucher_performance | Per voucher/campaign | 🆕 Build | W3 | Marketing ROI |
| - | gold_forecast_trends | Future predictions | 🆕 Build | W3 | ML predictions |
| - | gold_profit_loss | Daily per station | ⏸️ Deferred | FinOps | P&L with costs |

---

## Detailed Table Specifications

### 1. gold_daily_revenue ✅
**Purpose**: Core revenue and cash flow metrics
**Grain**: 1 row per station per day
**Rows**: ~3,758
**Columns**: 39

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `report_date`, `attributed_station_id`, `station_name`

**Volume**:
- `wash_count`, `unique_customers`, `membership_wash_count`, `prepaid_wash_count`

**Revenue**:
- `total_revenue` (service value), `total_cash_inflow` (actual cash)
- `wash_cash_inflow`, `topup_cash_inflow`, `sub_cash_inflow`
- `prepaid_usage`, `avg_wash_value`

**Top-ups**:
- `topup_count`, `topup_200k`, `topup_300k`, `topup_500k`, `topup_1m`, `topup_5m`

**Subscriptions**:
- `new_sub_count`, `new_sub_value`, `renewal_count`
- Breakdown by plan: `new_sub_silver_*`, `new_sub_gold_*`, `new_sub_vvip`

</details>

**Query Patterns**:
```sql
-- Revenue by station (date range)
SELECT * FROM gold_daily_revenue
WHERE report_date BETWEEN '2025-12-01' AND '2025-12-31'
  AND attributed_station_id = ?;

-- Network daily trend
SELECT report_date,
       SUM(total_revenue) as network_revenue,
       SUM(wash_count) as network_washes
FROM gold_daily_revenue
WHERE report_date >= '2025-01-01'
GROUP BY report_date
ORDER BY report_date;
```

---

### 2. gold_station_performance ✅
**Purpose**: Operational metrics with active user tracking
**Grain**: 1 row per station per day
**Rows**: ~8,046
**Columns**: 59

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `report_date`, `station_id`, `station_name`, `zone_name`

**Station Metadata**:
- `status`, `station_type`, `open_time`, `close_time`, `tags`

**Active Users** (rolling windows):
- `a30`, `a90` (active users 30/90 day)
- `s30`, `s90` (active subscribers 30/90 day)
- `non_sub_a30`, `non_sub_a90`

**Revenue Metrics**: (all 39 columns from gold_daily_revenue)

**Churn Metrics**:
- `user_churned_count`, `sub_churned_count`, `sub_at_risk_count`, `sub_reactivated_count`

</details>

**Query Patterns**:
```sql
-- Today's performance across all stations
SELECT * FROM gold_station_performance
WHERE report_date = CURRENT_DATE
ORDER BY total_revenue DESC;

-- Active user trend for one station
SELECT report_date, a30, s30, total_revenue
FROM gold_station_performance
WHERE station_id = ?
  AND report_date >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY report_date;
```

---

### 3. gold_subscription_mrr ⚠️
**Purpose**: Monthly Recurring Revenue tracking
**Grain**: 1 row per station per month
**Rows**: ~294
**Columns**: 23

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `report_month`, `attributed_station_id`, `station_name`

**New Subscriptions**:
- `new_sub_count`, `new_mrr`, `new_sub_revenue`

**Active (end-of-month snapshot)**:
- `active_sub_count`, `current_mrr`, `active_trial_count`
- `active_silver_count`, `active_gold_count`

**Churn**:
- `churned_count`, `at_risk_count`, `reactivated_count`

**Growth**:
- `mrr_growth_amount`, `mrr_growth_pct`

</details>

**Fixes Needed**:
1. Handle NULL `attributed_station_id` (11.8% unattributed)
2. Handle missing station names (14 rows)
3. Cap MRR growth rates (-100% to +500%)

**Query Patterns**:
```sql
-- MRR trend (last 12 months)
SELECT report_month,
       SUM(current_mrr) as total_mrr,
       SUM(active_sub_count) as total_active_subs
FROM gold_subscription_mrr
WHERE report_month >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '12 months')
  AND attributed_station_id IS NOT NULL
GROUP BY report_month
ORDER BY report_month;
```

---

### 4. gold_customer_segments 📝
**Purpose**: RFM segmentation and churn risk
**Grain**: 1 row per customer (snapshot)
**Rows**: ~393K
**Columns**: ~30

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `customer_id`, `full_name`, `phone`, `email`

**Activity**:
- `total_washes`, `lifetime_spend`, `first_wash_date`, `last_wash_date`
- `days_since_last_wash`, `stations_visited`, `washes_30d`

**RFM Scores** (1-5 scale):
- `recency_score` (5=recent, 1=long ago)
- `frequency_score` (5=frequent, 1=rare)
- `monetary_score` (5=high spend, 1=low spend)

**Segmentation**:
- `customer_segment`: 'VIP', 'Regular', 'Occasional', 'New', 'Inactive', 'Subscriber'
- `churn_risk`: 'Low', 'Medium', 'High', 'Churned'

</details>

**Action**: Rename `.disabled` to `.sql` and run dbt

**Query Patterns**:
```sql
-- VIP customers list
SELECT * FROM gold_customer_segments
WHERE customer_segment = 'VIP'
ORDER BY lifetime_spend DESC;

-- At-risk customers (for re-engagement campaign)
SELECT customer_id, full_name, phone,
       lifetime_spend, days_since_last_wash, churn_risk
FROM gold_customer_segments
WHERE churn_risk IN ('medium', 'high')
  AND lifetime_spend > 500000
ORDER BY lifetime_spend DESC;
```

---

### 5. gold_hourly_usage 📝
**Purpose**: Peak hours and capacity planning
**Grain**: 1 row per station per hour per day-of-week
**Rows**: ~2,000 (17 stations × 24 hours × 7 days)
**Columns**: ~20

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `station_id`, `station_name`, `order_hour`, `order_day_of_week`, `day_name`

**Metrics** (90-day rolling):
- `wash_count`, `revenue`, `avg_duration`

**Comparisons**:
- `network_total_washes`, `network_avg`, `pct_vs_network_avg`

**Peak Flags**:
- `is_peak_hour` (top 3 busiest hours overall)
- `is_day_peak` (busiest hour for this day of week)
- `utilization_level`: 'Very High', 'High', 'Medium', 'Low', 'Very Low'

</details>

**Action**: Rename `.disabled` to `.sql` and run dbt

**Query Patterns**:
```sql
-- Peak hours for one station
SELECT order_hour, day_name, wash_count
FROM gold_hourly_usage
WHERE station_id = ?
  AND is_peak_hour = TRUE
ORDER BY wash_count DESC;

-- Off-peak hours (for pricing optimization)
SELECT order_hour, AVG(wash_count) as avg_washes
FROM gold_hourly_usage
WHERE station_id = ?
  AND utilization_level IN ('Low', 'Very Low')
GROUP BY order_hour
ORDER BY avg_washes ASC;
```

---

### 6. gold_live_snapshot 🆕
**Purpose**: Real-time dashboard metrics
**Grain**: 1 row per station (current state)
**Refresh**: Every 5-15 minutes
**Columns**: ~35

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `station_id`, `station_name`, `snapshot_time`

**Today's Metrics**:
- `today_revenue`, `today_washes`, `today_customers`, `today_topups`, `today_new_subs`

**Yesterday Comparison**:
- `yesterday_revenue`, `revenue_change_pct`, `washes_change_pct`

**Current Active**:
- `current_a30`, `current_s30`, `active_subscriptions`, `at_risk_subs`

**Wallet**:
- `total_wallet_balance`, `customers_with_balance`

**Status**:
- `station_status`, `hours_since_last_wash`

</details>

**Implementation**: Scheduled Dagster job (every 5-15 min)

**Query Patterns**:
```sql
-- Live dashboard for all stations
SELECT * FROM gold_live_snapshot
WHERE snapshot_time >= CURRENT_TIMESTAMP - INTERVAL '5 minutes'
ORDER BY today_revenue DESC;
```

---

### 7. gold_kpi_summary 🆕
**Purpose**: Executive KPIs and growth metrics
**Grain**: Multiple views (network, monthly, station)
**Columns**: ~40 per view

#### View 1: gold_kpi_network (1 row - network total)

<details>
<summary><b>Key Columns</b></summary>

**All-Time**:
- `total_revenue`, `total_cash_inflow`, `total_washes`, `total_customers`, `total_new_subs`

**Month-to-Date (MTD)**:
- `mtd_revenue`, `mtd_washes`, `mtd_customers`, `mtd_new_subs`
- `mtd_revenue_vs_last_month_pct` (growth %)

**Year-to-Date (YTD)**:
- `ytd_revenue`, `ytd_washes`, `ytd_new_subs`
- `ytd_revenue_vs_last_year_pct`

**Key Ratios**:
- `arpu` (average revenue per user)
- `avg_wash_value`
- `conversion_rate` (non-sub → sub)
- `churn_rate_monthly`
- `customer_ltv` (estimated)

**Current State**:
- `active_stations`, `active_a30`, `active_s30`, `active_subscriptions`, `current_mrr`

</details>

#### View 2: gold_kpi_monthly (1 row per month)

<details>
<summary><b>Key Columns</b></summary>

- `report_month`
- `monthly_revenue`, `monthly_washes`, `monthly_active_customers`, `monthly_new_customers`
- `mom_revenue_growth_pct`, `yoy_revenue_growth_pct`
- `eom_active_subs`, `eom_mrr`, `eom_a30`, `eom_wallet_liability`

</details>

#### View 3: gold_kpi_station_summary (1 row per station)

<details>
<summary><b>Key Columns</b></summary>

- `station_id`, `station_name`, `zone_name`
- All-time: `total_revenue`, `total_washes`, `unique_customers`
- Averages: `avg_daily_revenue`, `avg_daily_washes`, `avg_wash_value`
- Current: `current_a30`, `current_s30`, `active_subscriptions`
- Rankings: `revenue_rank`, `washes_rank`, `customer_satisfaction_score`

</details>

**Query Patterns**:
```sql
-- Executive dashboard
SELECT * FROM gold_kpi_network;

-- Monthly trend (last 12 months)
SELECT * FROM gold_kpi_monthly
WHERE report_month >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '12 months')
ORDER BY report_month DESC;

-- Station rankings
SELECT * FROM gold_kpi_station_summary
ORDER BY total_revenue DESC;
```

---

### 8. gold_vehicle_analytics 🆕
**Purpose**: Vehicle tracking and fleet analysis
**Grain**: 1 row per vehicle
**Rows**: ~31K
**Columns**: ~25

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `vehicle_id`, `owner_id`, `owner_name`, `license_plate`

**Vehicle Details**: `vehicle_type`, `vehicle_brand`, `vehicle_color`

**Activity**:
- `total_washes`, `first_wash_date`, `last_wash_date`, `days_since_last_wash`
- `preferred_station_id`, `stations_visited`

**Financial**:
- `lifetime_spend`, `avg_wash_value`, `total_prepaid_usage`, `washes_30d`

**Status**:
- `vehicle_status`: 'active', 'inactive', 'churned'
- `is_primary_vehicle`

</details>

**Query Patterns**:
```sql
-- Inactive vehicles (re-engagement)
SELECT * FROM gold_vehicle_analytics
WHERE days_since_last_wash > 90
  AND lifetime_spend > 500000
ORDER BY lifetime_spend DESC;

-- Fleet customers (B2B opportunity)
SELECT owner_id, owner_name,
       COUNT(*) as vehicle_count,
       SUM(lifetime_spend) as total_spend
FROM gold_vehicle_analytics
GROUP BY owner_id, owner_name
HAVING COUNT(*) >= 3
ORDER BY total_spend DESC;
```

---

### 9. gold_station_summary 🆕
**Purpose**: All-time station totals and rankings
**Grain**: 1 row per station
**Rows**: ~17
**Columns**: ~35

<details>
<summary><b>Key Columns</b></summary>

**Time Range**:
- `first_activity_date`, `last_activity_date`, `days_operational`

**All-Time Volume**:
- `total_washes`, `total_customers`, `total_orders`, `total_transactions`

**All-Time Financial**:
- `total_revenue`, `total_cash_inflow` (with breakdown)
- `total_new_subs`, `total_renewals`

**Top-ups**: `total_topups`, breakdown by amount (200k, 300k, 500k, 1m, 5m)

**Averages**:
- `avg_daily_revenue`, `avg_daily_washes`, `avg_wash_value`
- `membership_penetration_pct`, `prepaid_usage_pct`

**Current**: `current_a30`, `current_s30`, `current_active_subs`, `current_wallet_balance`

</details>

**Query Patterns**:
```sql
-- Top performing stations
SELECT * FROM gold_station_summary
WHERE status = 'active'
ORDER BY total_revenue DESC
LIMIT 10;

-- Underperforming stations
SELECT station_name, avg_daily_revenue, days_operational
FROM gold_station_summary
WHERE days_operational > 30
ORDER BY avg_daily_revenue ASC;
```

---

### 10. gold_wash_mode_performance 🆕
**Purpose**: Product mix and service analysis
**Grain**: 1 row per station per month per wash_mode
**Rows**: ~2,000
**Columns**: ~20

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `report_month`, `station_id`, `mode_id`, `mode_name`, `mode_category`

**Volume**: `wash_count`, `unique_customers`, `membership_wash_count`

**Revenue**:
- `total_revenue`, `avg_wash_value`
- `mode_revenue_share_pct` (% of station's revenue)
- `mode_penetration_pct` (% of station's washes)

**Duration & Efficiency**:
- `avg_duration_minutes`, `washes_per_hour` (throughput)

**Growth**: `mom_wash_count_growth_pct`, `mom_revenue_growth_pct`

</details>

**Query Patterns**:
```sql
-- Product mix by station (last month)
SELECT station_name, mode_name,
       wash_count, total_revenue,
       mode_revenue_share_pct
FROM gold_wash_mode_performance
WHERE report_month = DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
ORDER BY station_name, mode_revenue_share_pct DESC;

-- Most efficient modes (revenue per hour)
SELECT mode_name,
       SUM(total_revenue) / SUM(total_duration_hours) as revenue_per_hour
FROM gold_wash_mode_performance
WHERE report_month >= '2025-01-01'
GROUP BY mode_name
ORDER BY revenue_per_hour DESC;
```

---

### 11. gold_customer_behavior 🆕 ⭐
**Purpose**: Customer 360 view for composable CRM
**Grain**: 1 row per customer (comprehensive)
**Rows**: ~393K
**Columns**: ~60

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**: `customer_id`, `full_name`, `phone`, `email`, `registered_at`

**Activity**:
- `first_wash_date`, `last_wash_date`, `days_since_last_wash`, `total_washes`
- `avg_days_between_washes`, `wash_frequency_category`
- `washes_last_30d`, `washes_last_90d`, `washes_last_365d`

**Financial**:
- `lifetime_spend`, `lifetime_revenue`, `avg_order_value`, `largest_order_value`

**Preferences**:
- `preferred_station_id`, `station_loyalty_pct`
- `preferred_mode_id`, `premium_mode_usage_pct`
- `preferred_payment_method`

**Payment**:
- `wallet_topup_count`, `avg_topup_amount`, `current_wallet_balance`

**Subscription**:
- `active_subscription_count`, `current_subscription_type`
- `subscription_start_date`, `days_until_renewal`, `subscription_lifetime_value`

**Vehicle**: `vehicle_count`, `primary_vehicle_type`, `has_multiple_vehicles`

**Marketing**:
- `voucher_redemption_count`, `responds_to_vouchers`, `last_campaign_interaction`

**RFM**: `recency_score`, `frequency_score`, `monetary_score`, `rfm_segment`

**Churn & Risk**:
- `churn_risk`, `churn_probability`, `at_risk_date`

**Predictions** (if ML available):
- `predicted_ltv`, `predicted_next_wash_date`, `predicted_churn_date`
- `propensity_to_subscribe`

**Cross-sell**:
- `cross_sell_score`, `recommended_action`

**Survey Data** (if integrated):
- `satisfaction_score`, `nps_score`, `satisfaction_category`

</details>

**Query Patterns**:
```sql
-- Customer 360 (CRM view)
SELECT * FROM gold_customer_behavior
WHERE customer_id = ?;

-- At-risk VIP customers (retention campaign)
SELECT customer_id, full_name, phone,
       lifetime_spend, days_since_last_wash, recommended_action
FROM gold_customer_behavior
WHERE rfm_segment IN ('VIP', 'Regular')
  AND churn_risk IN ('medium', 'high')
  AND days_since_last_wash < 180
ORDER BY lifetime_spend DESC
LIMIT 100;

-- Subscription upsell targets
SELECT customer_id, full_name, phone,
       total_washes, avg_days_between_washes, lifetime_spend
FROM gold_customer_behavior
WHERE active_subscription_count = 0
  AND washes_last_90d >= 6
  AND churn_risk = 'low'
ORDER BY propensity_to_subscribe DESC
LIMIT 50;

-- Detractors (low satisfaction follow-up)
SELECT customer_id, full_name, phone,
       satisfaction_score, nps_score, lifetime_spend
FROM gold_customer_behavior
WHERE satisfaction_category = 'detractor'
  AND last_survey_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY lifetime_spend DESC;
```

---

### 12. gold_cohort_retention 🆕
**Purpose**: Cohort analysis and LTV forecasting
**Grain**: 1 row per cohort month × retention month
**Rows**: ~500 (10 cohorts × 12 months × 4 stations)
**Columns**: ~25

<details>
<summary><b>Key Columns</b></summary>

**Identifiers**:
- `cohort_month` (acquisition month)
- `retention_month` (0, 1, 2, 3, ..., 12)
- `report_month` (actual calendar month)

**Cohort**:
- `cohort_size` (customers acquired)
- `acquisition_channel` (if tracked)

**Retention**:
- `retained_customers`, `retention_rate_pct`
- `churned_customers`, `churn_rate_pct`

**Activity**:
- `total_washes`, `washes_per_customer`, `active_customers`

**Revenue**:
- `total_revenue`, `revenue_per_customer`
- `cumulative_revenue`, `cumulative_ltv` (LTV progression)

**Subscriptions**:
- `subscribed_customers`, `subscription_rate_pct`, `subscription_mrr`

**Growth**: `mom_retention_change`, `mom_revenue_change`

</details>

**Query Patterns**:
```sql
-- Cohort retention curve
SELECT cohort_month, retention_month,
       cohort_size, retained_customers,
       retention_rate_pct, cumulative_ltv
FROM gold_cohort_retention
WHERE cohort_month >= '2025-01-01'
ORDER BY cohort_month, retention_month;

-- Month 3 retention benchmark
SELECT cohort_month,
       cohort_size,
       retention_rate_pct as month3_retention,
       cumulative_ltv as ltv_at_month3
FROM gold_cohort_retention
WHERE retention_month = 3
ORDER BY cohort_month DESC;
```

---

### 13. gold_wallet_economy 🆕
**Purpose**: Prepaid wallet analysis and liability tracking
**Grain**: Multiple views
**Columns**: ~20 per view

#### View 1: gold_wallet_summary (1 row per customer)

<details>
<summary><b>Key Columns</b></summary>

- `customer_id`, `customer_name`, `phone`
- `current_balance`, `last_topup_date`, `last_usage_date`, `last_activity_date`
- `total_topups`, `total_topup_amount`, `total_usage_amount`
- `avg_topup_amount`, `topup_frequency_days`
- `days_since_last_activity`
- `wallet_status`: 'active', 'dormant', 'zero_balance', 'churned'

</details>

#### View 2: gold_wallet_trends (1 row per month)

<details>
<summary><b>Key Columns</b></summary>

- `report_month`
- `total_wallet_balance` (liability)
- `customers_with_balance`, `avg_balance_per_customer`
- `topup_count`, `topup_amount`, `usage_amount`
- `topup_vs_usage_ratio`, `wallet_utilization_rate`
- `dormant_wallet_balance` (>90 days inactive)
- `dormant_customer_count`

</details>

**Query Patterns**:
```sql
-- Dormant wallets (re-engagement opportunity)
SELECT * FROM gold_wallet_summary
WHERE current_balance > 50000
  AND days_since_last_activity > 90
ORDER BY current_balance DESC;

-- Wallet liability trend
SELECT report_month,
       total_wallet_balance,
       customers_with_balance,
       dormant_wallet_balance
FROM gold_wallet_trends
WHERE report_month >= '2025-01-01'
ORDER BY report_month;
```

---

## Week 3 & Future Tables

### gold_voucher_performance 🆕 (Week 3)
**Purpose**: Marketing campaign ROI
**Grain**: 1 row per voucher or campaign
**Columns**: ~20

**Key Metrics**:
- Redemption rate, revenue attributed, discount given
- Customer acquisition, ROI (revenue - discount)
- New vs existing customer usage

---

### gold_forecast_trends 🆕 (Week 3)
**Purpose**: ML prediction storage
**Grain**: 1 row per station per future month
**Columns**: ~15

**Predictions**:
- Revenue forecast (next 3/6/12 months)
- Customer growth forecast, churn forecast
- Confidence intervals

---

### gold_profit_loss ⏸️ (Future - FinOps)
**Purpose**: P&L with full cost breakdown
**Grain**: 1 row per station per day
**Columns**: ~30

**Metrics**:
- Reconciled revenue (operations + bank matched)
- COGS (water, electricity, chemicals, depreciation)
- OpEx (staff, rent, maintenance, marketing)
- Margins (gross profit, operating profit, %)
- Unit economics (cost per wash, margin per wash)

---

## Data Flow Architecture

```
┌────────────────────────────────────────────────────────┐
│ Source Systems (PostgreSQL)                            │
│ - Operations DB (orders, washes, transactions)        │
│ - Users DB (customers, vehicles, subscriptions)       │
└────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────┐
│ Bronze Layer (Iceberg in MinIO S3)                    │
│ - 20+ raw tables (append-only, permanent)            │
└────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────┐
│ Silver Layer (DuckDB - transform/lakehouse.duckdb)    │
│ - Dimensions: 8 tables (customers, vehicles, etc)     │
│ - Facts: 5 tables (washes, transactions, etc)         │
└────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────┐
│ Gold Layer (DuckDB - same file)                       │
│ - 13 business-ready analytical tables                  │
│ → Single source of truth for all downstream systems   │
└────────────────────────────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────┐
│ Downstream Systems                                      │
│ - Analytics App (datamart API)                         │
│ - Composable CRM (customer 360)                       │
│ - Metabase/PowerBI (dashboards)                       │
│ - ML Pipeline (feature store)                          │
└────────────────────────────────────────────────────────┘
```

---

## Query Architecture

### Query Organization

```
queries/
├── gold/
│   ├── revenue/
│   │   ├── daily_revenue_by_station.sql
│   │   ├── monthly_revenue_trend.sql
│   │   └── revenue_vs_target.sql
│   ├── customers/
│   │   ├── customer_360_view.sql
│   │   ├── high_value_customers.sql
│   │   ├── churn_risk_list.sql
│   │   └── rfm_segments.sql
│   ├── stations/
│   │   ├── station_rankings.sql
│   │   ├── station_health_check.sql
│   │   └── underperforming_stations.sql
│   ├── subscriptions/
│   │   ├── mrr_trend.sql
│   │   ├── subscription_cohorts.sql
│   │   └── at_risk_subscribers.sql
│   └── operations/
│       ├── live_dashboard.sql
│       ├── kpi_scorecard.sql
│       └── hourly_capacity.sql
```

### Query Access Patterns

**1. Analytics App (Datamart API)**:
```python
# Generate SQL queries from gold tables
query = load_query("gold/operations/live_dashboard.sql")
metrics = duckdb_conn.execute(query).fetchdf()
return JSONResponse(metrics.to_dict('records'))
```

**2. Composable CRM**:
```python
# Customer 360 view
query = load_query("gold/customers/customer_360_view.sql",
                  customer_id=user_id)
profile = duckdb_conn.execute(query).fetchone()
return CustomerProfile(**profile)
```

**3. PowerBI/Metabase**:
- Direct DuckDB ODBC connection
- Query gold tables with saved SQL
- Build dashboards on gold layer

**4. ML Pipeline**:
```python
# Gold tables as feature store
features = duckdb_conn.execute("""
    SELECT customer_id,
           total_washes, lifetime_spend, days_since_last_wash,
           avg_days_between_washes, churn_risk
    FROM gold_customer_behavior
    WHERE total_washes >= 3
""").fetchdf()

# Train model
model.fit(features[feature_cols], features['churned'])

# Write predictions back to gold
predictions_df.to_sql('gold_customer_predictions', duckdb_conn)
```

---

## CRM Use Cases (B2C Focus)

### Customer Satisfaction & Retention
- **Track satisfaction scores**: NPS, CSAT from surveys
- **Identify detractors**: Low satisfaction + high lifetime value → priority follow-up
- **At-risk detection**: Churn probability > 70% → retention campaign
- **Win-back campaigns**: Churned customers with high LTV

### Targeted Marketing
- **Segmented campaigns**: Different messaging for VIP vs Occasional customers
- **Voucher targeting**: Responds_to_vouchers = true → send promotion
- **Upsell opportunities**: High frequency + no subscription → offer membership
- **Cross-sell**: Multiple vehicles → suggest family plan

### Customer Journey Optimization
- **Cohort analysis**: Which acquisition channels have best retention?
- **Frequency optimization**: Avg days between washes → reminder timing
- **Wallet engagement**: Low balance + high usage → topup nudge
- **Mode preferences**: Basic mode users → upsell to premium

### Behavioral Insights
- **Station loyalty**: High loyalty % → station-specific offers
- **Payment preferences**: Wallet vs card → optimize payment flow
- **Vehicle patterns**: Primary vehicle vs secondary → targeted campaigns
- **Seasonality**: Wash frequency by month → predict demand

---

## Implementation Checklist

### Week 1: Foundation + Analytics & Insights
- [ ] Fix gold_subscription_mrr (attribution, station names, growth caps)
- [ ] Enable gold_customer_segments (rename .disabled → .sql)
- [ ] Enable gold_hourly_usage (rename .disabled → .sql)
- [ ] Build gold_live_snapshot (dbt model + Dagster refresh job)
- [ ] Build gold_kpi_summary (3 views: network, monthly, station)
- [ ] Build gold_vehicle_analytics
- [ ] Build gold_station_summary
- [ ] Build gold_wash_mode_performance
- [ ] Create SQL queries for each table in /queries/gold/
- [ ] Test query performance (<1 second for most queries)
- [ ] Verify data quality (row counts, NULLs, aggregations)

### Week 2: CRM Foundation
- [ ] Build gold_customer_behavior (60+ columns, comprehensive 360 view)
- [ ] Build gold_cohort_retention (retention curves + LTV)
- [ ] Build gold_wallet_economy (2 views: customer + trends)
- [ ] Design CRM query patterns (customer_360_view.sql, churn_risk_list.sql, etc.)
- [ ] Document CRM data model (customer lifecycle, scoring)
- [ ] Create CRM-specific queries in /queries/gold/customers/
- [ ] Test CRM integration (API endpoints, query performance)

### Week 3: Advanced Analytics & Marketing
- [ ] Build gold_voucher_performance (marketing ROI)
- [ ] Build gold_forecast_trends_staging (ML predictions storage)
- [ ] Setup ML pipeline foundation (Python scripts, feature engineering)
- [ ] Document ML pipeline architecture
- [ ] Create example ML models (churn prediction, LTV prediction)

---

## Success Criteria

After implementation:
- ✅ All 11 analytics app features have data sources (P&L deferred)
- ✅ CRM foundation complete (customer_behavior + segments + cohorts)
- ✅ Live dashboard functional (real-time metrics)
- ✅ Executive KPIs available (network, monthly, station views)
- ✅ All gold tables have SQL queries in /queries/
- ✅ Query performance <1 second for 95% of queries
- ✅ Data quality validated (row counts, NULLs, totals match silver)
- ✅ CLI commands working (jetx-lake gold <command>)
- ✅ ML pipeline foundation ready (gold tables as feature store)

---

## Future Enhancements (FinOps Integration)

When FinOps system is ready (~1 month):

### New Silver Tables
- `fct_costs` (transaction-level costs)
- `fct_revenue_reconciled` (operations + bank matched)
- `dim_cost_centers` (stations, departments)

### New Gold Tables
- `gold_profit_loss` (daily P&L by station with full cost breakdown)
- `gold_unit_economics` (cost per wash, margin by service type)
- `gold_budget_vs_actual` (variance analysis)

### Integration Flow
```
FinOps (Soft Ledger)
  → Near real-time API/Parquet
  → Bronze (transaction-level costs + bank reconciliation)
  → Silver (fct_costs, fct_revenue_reconciled)
  → Gold (P&L aggregates)
```

---

**Document Version**: 1.0
**Last Updated**: 2026-01-29
**Status**: Ready for Week 1 Implementation
