# Session Log - 2026-01-14

## Summary
Fixed critical aggregation issues in MRR reporting and churn metrics. The problem was that daily snapshot data from `fct_daily_churn` was being incorrectly SUMMed across days, causing double-counting of subscribers.

## Issues Fixed

### 1. Churn Metrics Double-Counting
**Problem**: MRR report showed "At-risk: 2,708" when the actual number should be 125.

**Root Cause**:
- `fct_daily_churn` contains daily **snapshots** (point-in-time counts), not events
- Original code was SUMMing these snapshots across all days in a month
- This counted the same subscriber multiple times (once for each day they appeared)

**Fix Applied**:
- **File**: `transform/models/gold/gold_subscription_mrr.sql` (lines 161-184)
- Changed from `SUM(sub_churned_count)` to using latest day's snapshot
- Added `monthly_churn_latest` CTE with `ROW_NUMBER()` to get most recent date per month
- Modified `monthly_churn` to select only `WHERE rn = 1`

**Code Changes**:
```sql
-- Before (INCORRECT - summed snapshots)
monthly_churn AS (
    SELECT
        DATE_TRUNC('month', report_date) as churn_month,
        station_id as attributed_station_id,
        SUM(sub_churned_count) as churned_count,
        SUM(sub_at_risk_count) as at_risk_count,
        SUM(sub_reactivated_1_count + sub_reactivated_2_count) as reactivated_count
    FROM {{ ref('fct_daily_churn') }}
    GROUP BY DATE_TRUNC('month', report_date), station_id
)

-- After (CORRECT - latest snapshot)
monthly_churn_latest AS (
    SELECT
        DATE_TRUNC('month', report_date) as churn_month,
        station_id as attributed_station_id,
        report_date,
        sub_churned_count,
        sub_at_risk_count,
        sub_reactivated_1_count,
        sub_reactivated_2_count,
        ROW_NUMBER() OVER (PARTITION BY DATE_TRUNC('month', report_date), station_id
                          ORDER BY report_date DESC) as rn
    FROM {{ ref('fct_daily_churn') }}
),

monthly_churn AS (
    SELECT
        churn_month,
        attributed_station_id,
        sub_churned_count as churned_count,
        sub_at_risk_count as at_risk_count,
        sub_reactivated_1_count + sub_reactivated_2_count as reactivated_count
    FROM monthly_churn_latest
    WHERE rn = 1
)
```

### 2. CLI Network-Level Aggregation Issue
**Problem**: Even after fixing the SQL, CLI still showed 250 instead of 125.

**Root Cause**:
- `gold_subscription_mrr` contains BOTH station-level AND network-level rows (station_id IS NULL)
- CLI was doing `SUM()` without filtering, adding both network total AND individual stations

**Fix Applied**:
- **File**: `cli/commands/gold.py` (lines 595-605, 617, 620)
- Added `attributed_station_id IS NULL` filter when showing network totals
- Prevents double-counting by only using pre-aggregated network row

**Code Changes**:
```python
# Before (INCORRECT - summed all rows)
where_parts = []
params = []
if station_id:
    where_parts.append("attributed_station_id = ?")
    params.append(station_id)
where_clause = f"WHERE {' AND '.join(where_parts)}" if where_parts else ""

# After (CORRECT - filters for network total)
where_parts = []
params = []
if station_id:
    where_parts.append("attributed_station_id = ?")
    params.append(station_id)
else:
    # For network total, only include network-level aggregate
    where_parts.append("attributed_station_id IS NULL")
where_clause = f"WHERE {' AND '.join(where_parts)}"
```

## Verification

### Before Fix:
```
Subscriber Activity (This Month)
  Churned:      0
  At-risk:      2,708  ← WRONG (double-counted)
  Reactivated:  2,230  ← WRONG (double-counted)
```

### After Fix:
```
Subscriber Activity (This Month)
  Churned:      0
  At-risk:      125   ← CORRECT (matches latest snapshot)
  Reactivated:  107   ← CORRECT (matches latest snapshot)
```

### Validation Queries:
```sql
-- Check daily snapshots for Jan 2026
SELECT report_date, sub_at_risk_count, sub_reactivated_1_count + sub_reactivated_2_count
FROM main_silver.fct_daily_churn
WHERE report_date = '2026-01-14' AND station_id IS NULL;
-- Result: at_risk=125, reactivated=107 ✓

-- Check gold model output
SELECT at_risk_this_month, reactivated_this_month
FROM main_gold.gold_subscription_mrr
WHERE subscription_month = '2026-01-01' AND attributed_station_id IS NULL;
-- Result: at_risk=125, reactivated=107 ✓
```

## Files Modified

1. **transform/models/gold/gold_subscription_mrr.sql**
   - Lines 161-184: Fixed churn aggregation logic
   - Changed from SUM to latest snapshot using ROW_NUMBER()

2. **cli/commands/gold.py**
   - Lines 595-605: Added network-level filter logic
   - Line 617: Updated summary query WHERE clause
   - Line 620: Updated to use IS NULL filter for network totals

## Technical Details

### Understanding Daily Snapshots vs Events
- **Snapshot**: Point-in-time count (e.g., "125 subscribers are at-risk on 2026-01-14")
- **Event**: Something that happened (e.g., "10 subscribers churned on 2026-01-14")

`fct_daily_churn` contains **snapshots**, which means:
- ✓ Can use latest value for current state
- ✓ Can calculate changes between dates
- ✗ Cannot SUM across dates (double-counts same entities)

### Network vs Station Aggregation
The gold layer stores data at multiple grain levels:
- `attributed_station_id IS NULL` → Network-wide total
- `attributed_station_id = <uuid>` → Individual station

When querying for network totals, must filter for NULL to avoid summing network + all stations.

## Impact
- MRR report now shows correct churn metrics
- Churn numbers match the underlying daily snapshot data
- Network-level and station-level reports work correctly
- No data loss or incorrect business logic

## Commands to Rebuild
```bash
# Rebuild gold layer
cd transform
dbt run --select gold_subscription_mrr

# Test MRR report
python -m cli.main gold mrr
```

## Related Files
- Source: `transform/models/silver/fct_daily_churn.sql` (churn logic definitions)
- Source: `transform/models/silver/fct_daily_active_users.sql` (A30/A90/S30/S90 metrics)
- Model: `transform/models/gold/gold_subscription_mrr.sql` (MRR aggregation)
- CLI: `cli/commands/gold.py` (reporting commands)

## Notes
- Previous session completed: gold_daily_revenue fixes, MRR historical calculation
- This session: Fixed churn aggregation issues discovered during testing
- All gold layer models now working correctly with proper aggregation logic
