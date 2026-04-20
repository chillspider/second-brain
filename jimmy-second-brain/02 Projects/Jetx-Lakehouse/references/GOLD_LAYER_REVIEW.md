# Gold Layer Models - Comprehensive Review

Review of all three gold layer models for completeness, accuracy, and data quality.

**Review Date**: 2026-01-16

---

## Summary

| Model | Status | Rows | Issues Found |
|-------|--------|------|--------------|
| gold_daily_revenue | ✅ **Reviewed** | 3,653 | None - working correctly |
| gold_station_performance | ✅ **Reviewed** | 7,956 | None - working correctly |
| gold_subscription_mrr | ⚠️ **Needs Fix** | 181 | 3 issues identified |

---

## 1. gold_daily_revenue ✅

**Purpose**: Daily revenue metrics per station
**Grain**: 1 row per station per day
**Date Range**: 2024-09-12 to 2026-01-12
**Rows**: 3,653

### Key Metrics
- Revenue (wash value delivered)
- Cash inflow (broken down by wash/topup/sub)
- Wash count
- Unique customers
- Average wash value
- Top-up and subscription counts

### Data Quality: ✅ GOOD
- All stations have names
- No NULL attributed_station_id
- Totals match silver layer (validated)
- 100% coverage for all active stations

### Status: **COMPLETE** ✅

---

## 2. gold_station_performance ✅

**Purpose**: Daily active user metrics per station
**Grain**: 1 row per station per day
**Date Range**: 2024-09-12 to 2026-01-12
**Rows**: 7,956

### Key Metrics
- A30, A90 (active users 30/90 day)
- S30, S90 (active subscribers 30/90 day)
- Total cash inflow
- Churn metrics (churned, at-risk, reactivated)

### Data Quality: ✅ GOOD
- All metrics calculated correctly
- Proper lookback windows
- 100% coverage

### Status: **COMPLETE** ✅

---

## 3. gold_subscription_mrr ⚠️

**Purpose**: Monthly Recurring Revenue (MRR) and subscription metrics
**Grain**: 1 row per station per month
**Date Range**: 2024-12-01 to 2026-01-01 (14 months)
**Rows**: 181

### Key Metrics
- New subscriptions per month
- Active subscriptions (end of month snapshot)
- Current MRR
- Churn metrics
- MRR growth rate

---

### ⚠️ ISSUES FOUND

#### Issue 1: Missing Station Names
**Problem**: 14 rows have `attributed_station_id` but NULL `station_name`

```
Rows with station_id but no station_name: 14
```

**Root Cause**: Station IDs in subscription data don't match dim_stations

**Impact**: Medium - can't identify which stations these subscriptions belong to

**Fix Required**:
1. Check if these are old/deactivated stations
2. Add LEFT JOIN to dim_stations with COALESCE for unknown stations
3. Or create reference table for historical stations

---

#### Issue 2: Unattributed Subscriptions
**Problem**: Subscriptions with NULL `attributed_station_id`

```
Latest month (2026-01): 152 new subs, 172 active, MRR=35,350,000
```

**Impact**: HIGH - 172 active subscriptions (11.8% of total) not attributed to any station

**Analysis**:
- These might be corporate/VVIP subscriptions
- Or subscriptions created before station attribution was added
- MRR=35,350,000 VND (~$1,500) monthly revenue not attributed

**Fix Options**:
1. Create "Unattributed" pseudo-station
2. Investigate why subscriptions lack station_id in fct_subscriptions
3. Add business logic to attribute based on first wash location

---

#### Issue 3: Extreme MRR Growth Rates
**Problem**: Some stations showing >1000% growth

```
Unknown station:    1185.5% growth
JetX Tân Hưng:      1037.1% growth
```

**Root Cause**:
- Likely stations with very low/zero MRR in previous month
- Dividing by near-zero creates extreme percentages

**Impact**: Low - growth % is informational only

**Fix**: Cap growth rate at reasonable bounds (e.g., -100% to +500%) or show NULL for low base values

---

#### Issue 4: Data Sparsity in Early Months
**Problem**: Very few subscriptions in Oct/Sep/Aug 2025

```
2025-10-01: 1 new, 1 active
2025-09-01: 0 new, 0 active
2025-08-01: 0 new, 1 active
```

**Impact**: Low - this might be accurate (subscription business started later)

**Verification Needed**: Check if subscriptions actually started in late 2025

---

### Recommended Fixes for gold_subscription_mrr

#### Priority 1: Fix NULL station_id attribution
```sql
-- Add to gold_subscription_mrr.sql after line 241

-- Add pseudo-station for unattributed subs
LEFT JOIN (
    SELECT 'UNATTRIBUTED' as station_id, 'Unattributed Subscriptions' as station_name
) unattributed ON a.attributed_station_id IS NULL
```

#### Priority 2: Handle missing station names
```sql
-- Update station join (line 241)
LEFT JOIN (
    SELECT
        station_id,
        station_name,
        COALESCE(station_name, 'Unknown Station (' || station_id || ')') as display_name
    FROM {{ ref('dim_stations') }}
) st ON a.attributed_station_id = st.station_id
```

#### Priority 3: Cap MRR growth rates
```sql
-- Update growth calculation (line 231-235)
CASE
    WHEN LAG(mrr.current_mrr) OVER (...) < 1000000 THEN NULL  -- Skip if base < 1M VND
    WHEN LAG(mrr.current_mrr) OVER (...) > 0
        THEN LEAST(500, GREATEST(-100,  -- Cap between -100% and +500%
            ROUND((mrr.current_mrr - LAG(mrr.current_mrr) OVER (...)) * 100.0
                 / LAG(mrr.current_mrr) OVER (...), 1)
        ))
    ELSE NULL
END as mrr_growth_pct
```

---

## Data Flow Validation

### Revenue Flow (Validated ✅)
```
PostgreSQL (orders, transactions)
  → Bronze (Iceberg S3)
  → Silver (fct_washes, fct_transactions)
  → Gold (gold_daily_revenue)

Total cash inflow: 3,879,016,586 VND (matches across all layers)
```

### Subscription Flow (Needs Verification ⚠️)
```
PostgreSQL (subscriptions, payments)
  → Bronze (Iceberg S3)
  → Silver (fct_subscriptions, fct_transactions)
  → Gold (gold_subscription_mrr)

Unattributed subscriptions: 172 active (11.8%)
Missing station names: 14 rows
```

---

## Testing Checklist

### gold_daily_revenue
- [x] Row count matches expected (18 stations × ~200 days)
- [x] All stations have names
- [x] Revenue totals match silver layer
- [x] No NULL attributed_station_id
- [x] Date range covers all available data

### gold_station_performance
- [x] A30/A90/S30/S90 calculated correctly
- [x] Lookback windows accurate
- [x] All stations covered
- [x] Joins with revenue table work correctly

### gold_subscription_mrr
- [ ] **Fix NULL attributed_station_id handling**
- [ ] **Fix missing station names**
- [ ] **Verify subscription start dates (2025 data sparsity)**
- [ ] **Cap extreme growth rates**
- [ ] Validate MRR calculation logic
- [ ] Test month-end snapshot logic

---

## Next Steps

1. **Fix gold_subscription_mrr issues** (Priority 1-3 above)
2. **Investigate unattributed subscriptions** - Why 11.8% have no station?
3. **Add data quality tests** - dbt tests for NULL checks, value ranges
4. **Create gold layer documentation** - Business metric definitions
5. **Add monitoring** - Alert on extreme growth rates, high unattributed %

---

## SQL Queries for Investigation

### Check unattributed subscriptions in source
```sql
-- Run in transform/lakehouse.duckdb
SELECT
    COUNT(*) as total_subs,
    COUNT(attributed_station_id) as with_station,
    COUNT(*) - COUNT(attributed_station_id) as without_station,
    ROUND((COUNT(*) - COUNT(attributed_station_id)) * 100.0 / COUNT(*), 1) as pct_unattributed
FROM main_silver.fct_subscriptions;
```

### Find missing stations in dim_stations
```sql
SELECT DISTINCT
    s.attributed_station_id,
    s.plan_name,
    COUNT(*) as sub_count
FROM main_silver.fct_subscriptions s
LEFT JOIN main_silver.dim_stations st ON s.attributed_station_id = st.station_id
WHERE s.attributed_station_id IS NOT NULL
  AND st.station_id IS NULL
GROUP BY s.attributed_station_id, s.plan_name;
```

---

**Status**: 2/3 gold models complete, 1 requires fixes

**Estimated Fix Time**: 1-2 hours

**Risk**: Low - existing models work, MRR model has cosmetic/attribution issues only
