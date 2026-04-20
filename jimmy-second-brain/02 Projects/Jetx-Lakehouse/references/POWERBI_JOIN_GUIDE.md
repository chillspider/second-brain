# PowerBI Join Guide - Gold Tables

## ✅ Data Validation Complete

The cash inflow data is **100% accurate** between tables:
- `gold_daily_revenue`: 3,879,016,586 total cash inflow
- `gold_station_performance`: 3,879,016,586 total cash inflow
- **Difference: 0**

## Column Name Differences (IMPORTANT!)

### gold_daily_revenue
- Station column: `attributed_station_id`
- Date column: `report_date`
- Grain: 1 row per day per station

### gold_station_performance
- Station column: `station_id` ⚠️ **Different name!**
- Date column: `report_date`
- Grain: 1 row per day per station

## How to Join in PowerBI

### Option 1: Direct Join (Recommended)

When creating relationships in PowerBI Model view:

```
gold_daily_revenue[attributed_station_id] → gold_station_performance[station_id]
gold_daily_revenue[report_date] → gold_station_performance[report_date]
```

**Cardinality**: Many-to-One (M:1)
**Cross filter direction**: Both

### Option 2: Power Query Merge

In Power Query Editor:

1. Select `gold_daily_revenue` table
2. Home → Merge Queries
3. Join columns:
   - Left: `attributed_station_id` = Right: `station_id`
   - Left: `report_date` = Right: `report_date`
4. Join Kind: **Inner** (to match only overlapping rows)

## Common Issues & Solutions

### Issue: Cash inflow shows much lower than expected

**Cause**: Incorrect filter or aggregation

**Solutions**:

1. **Check your date filter**:
   - Are you filtering to a specific date range?
   - Make sure both tables use the same date filter

2. **Check for duplicate joins**:
   - Joining on just station_id without date can cause duplicates
   - Always join on BOTH station_id AND report_date

3. **Verify table relationship**:
   - PowerBI Model view → Check relationship lines
   - Should be: `attributed_station_id ↔ station_id` AND `report_date ↔ report_date`

### Issue: Numbers don't match between tables

**Cause**: Different row counts

**Explanation**:
- `gold_daily_revenue`: 3,635 rows (only days with revenue)
- `gold_station_performance`: 7,956 rows (includes days with no revenue but has active users/churn)

**Solution**: Use INNER JOIN or filter to matching dates only

## Example DAX Measures

### Total Cash Inflow (from either table)

```dax
Total Cash Inflow =
SUM(gold_daily_revenue[total_cash_inflow])

-- OR (same result)
Total Cash Inflow =
SUM(gold_station_performance[total_cash_inflow])
```

### Cash Inflow by Station (after join)

```dax
Station Cash Inflow =
CALCULATE(
    SUM(gold_daily_revenue[total_cash_inflow]),
    ALLEXCEPT(gold_daily_revenue, gold_daily_revenue[station_name])
)
```

### Month-to-Date Cash

```dax
MTD Cash Inflow =
CALCULATE(
    SUM(gold_daily_revenue[total_cash_inflow]),
    DATESMTD(gold_daily_revenue[report_date])
)
```

## Validation Query (Run in DuckDB)

To verify your data before loading into PowerBI:

```sql
-- Check totals match
SELECT
    'gold_daily_revenue' as table_name,
    COUNT(*) as rows,
    SUM(total_cash_inflow) as total_cash
FROM main_gold.gold_daily_revenue
WHERE attributed_station_id IS NOT NULL

UNION ALL

SELECT
    'gold_station_performance' as table_name,
    COUNT(*) as rows,
    SUM(total_cash_inflow) as total_cash
FROM main_gold.gold_station_performance
WHERE station_id IS NOT NULL;
```

Expected result:
```
gold_daily_revenue         3,635 rows    3,879,016,586
gold_station_performance   7,956 rows    3,879,016,586
```

## Station Totals (All-Time)

| Station | Total Cash Inflow |
|---------|-------------------|
| JetX Lương Định Của | 1,347,486,308 |
| JetX Vinhomes Grand Park | 465,058,000 |
| JetX GO! Hạ Long | 344,480,277 |
| JetX Linh Đàm | 303,969,000 |
| JetX GO! Thăng Long | 278,100,999 |

These numbers are **identical** in both tables when properly joined.

## Troubleshooting Checklist

If your PowerBI report shows different numbers:

- [ ] Check relationship in Model view (attributed_station_id ↔ station_id)
- [ ] Verify date filters are applied to both tables
- [ ] Look for duplicate rows (join on BOTH station AND date)
- [ ] Check if VVIP/unattributed rows are excluded (attributed_station_id IS NOT NULL)
- [ ] Validate with the DuckDB query above before loading to PowerBI

## Need Help?

Run this diagnostic script:
```bash
python3 debug_cash_inflow.py
```

This will show:
- Total cash inflow in each table
- Per-station totals
- Any discrepancies

---

**Last Updated**: 2026-01-15
**Verified**: All cash inflow numbers match 100% between tables
