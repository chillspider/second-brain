# Metabase Setup Guide - JetX Lakehouse

## Quick Start

Access Metabase at: **http://localhost:3001**

## Database Configuration (Already Done)

Your Metabase is already configured with the DuckDB connection:

- **Database name**: JetX-LakeHouse-DuckDB
- **Database type**: DuckDB
- **Database file**: `:memory` (with attached `lakehouse.duckdb`)
- **Init SQL**: `docker/metabase/duckdb_init.sql`

## Available Tables/Views

### Gold Layer (For Dashboards) ⭐

Use these for business dashboards and reports:

1. **`gold_daily_revenue`** (3,653 rows)
   - Daily revenue metrics by station
   - Columns: report_date, station_id, revenue, wash_count, topups, subscriptions, etc.
   - Grain: 1 row = 1 day + 1 station
   - Use for: Daily/weekly/monthly revenue trends

2. **`gold_station_performance`** (59 columns)
   - Comprehensive daily station metrics
   - Includes: A30, A90, S30, S90, revenue, churn, wash volumes
   - Grain: 1 row = 1 day + 1 station
   - Use for: Station rankings, performance comparisons

3. **`gold_subscription_mrr`**
   - Monthly recurring revenue metrics
   - Columns: subscription_month, MRR, active_subscriptions, churn
   - Grain: 1 row = 1 month + 1 station
   - Use for: Subscription health, MRR trends

### Silver Layer (For Ad-hoc Analysis)

Available via convenience views:

- `dim_stations` - Station master data (17 stations)
- `dim_customers` - Customer master data
- `fct_daily_active_users` - Daily A30/A90/S30/S90 metrics
- `fct_daily_churn` - Daily churn snapshots

### Bronze Layer (For Raw Data Exploration)

Available as views reading from S3:

- `bronze_orders` - Raw order data
- `bronze_users` - Raw user data
- `bronze_subscriptions` - Raw subscription data
- (20+ bronze tables total)

## How to Refresh Data

### After Running dbt

When you run `dbt run` to update your data:

1. **Automatic Refresh** (Recommended):
   - Metabase reads from attached `lakehouse.duckdb`
   - Changes appear automatically on next query
   - No action needed!

2. **Force Refresh** (If tables not updating):
   ```bash
   docker-compose restart metabase
   ```

3. **Sync Schema** (If new tables added):
   - Go to: Settings → Admin → Databases
   - Click "Sync database schema now"

## Building Your First Dashboard

### Example 1: Daily Revenue Trend

1. Click "New" → "Question"
2. Select database: "JetX-LakeHouse-DuckDB"
3. Select table: `gold_daily_revenue`
4. Visualization: Line chart
5. X-axis: `report_date`
6. Y-axis: Sum of `revenue`
7. Filter: `attributed_station_id IS NULL` (for network total)

### Example 2: Station Performance Leaderboard

1. New Question → Select `gold_station_performance`
2. Visualization: Table
3. Columns: `station_name`, `zone_name`, sum(`wash_count`), sum(`revenue`)
4. Filter: `report_date >= DATE_TRUNC('month', CURRENT_DATE)`
5. Sort by: `revenue` DESC
6. Limit: 10

### Example 3: MRR Growth Chart

1. New Question → Select `gold_subscription_mrr`
2. Visualization: Line chart
3. X-axis: `subscription_month`
4. Y-axis: Sum of `current_mrr`
5. Filter: Last 12 months
6. Group by: `attributed_station_id` (optional, for breakdown)

## Recommended Dashboards

### 1. Executive Dashboard
- **Revenue**: Daily revenue trend (last 30 days)
- **Stations**: Top 10 stations by revenue
- **Subscriptions**: Active MRR, new subscriptions
- **Customers**: A30/A90 trends

### 2. Station Performance Dashboard
- **Leaderboard**: Station rankings (revenue, washes, customers)
- **Map view**: Revenue by zone
- **Time series**: Wash volume trends by station
- **Operational**: Avg wash duration, membership %

### 3. Subscription Health Dashboard
- **MRR Trend**: Historical MRR growth
- **Churn Metrics**: Churned, at-risk, reactivated
- **Cohort Analysis**: New subscriptions by month
- **Plan Mix**: Silver vs Gold distribution

### 4. Customer Analytics Dashboard
- **Active Users**: A30, A90, S30, S90 trends
- **Churn Analysis**: User churn vs subscriber churn
- **Reactivation**: User reactivation funnel
- **Segmentation**: By station, by subscription status

## SQL Query Examples

### Custom Query: Network Revenue Summary
```sql
SELECT
    DATE_TRUNC('month', report_date) as month,
    SUM(revenue) as total_revenue,
    SUM(wash_count) as total_washes,
    COUNT(DISTINCT attributed_station_id) as active_stations
FROM gold_daily_revenue
WHERE attributed_station_id IS NOT NULL
GROUP BY DATE_TRUNC('month', report_date)
ORDER BY month DESC
LIMIT 12
```

### Custom Query: Station MRR Contribution
```sql
SELECT
    s.station_name,
    s.zone_name,
    m.current_mrr,
    m.active_subscriptions,
    ROUND(m.current_mrr * 100.0 / SUM(m.current_mrr) OVER (), 2) as mrr_pct
FROM gold_subscription_mrr m
JOIN dim_stations s ON m.attributed_station_id = s.station_id
WHERE m.subscription_month = DATE_TRUNC('month', CURRENT_DATE)
    AND m.attributed_station_id IS NOT NULL
ORDER BY m.current_mrr DESC
```

### Custom Query: Station Performance Matrix
```sql
SELECT
    station_name,
    zone_name,
    SUM(wash_count) as total_washes,
    SUM(revenue) as total_revenue,
    MAX(a30) as latest_a30,
    MAX(s30) as latest_s30,
    ROUND(AVG(membership_pct), 1) as avg_membership_pct
FROM gold_station_performance
WHERE report_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY station_id, station_name, zone_name
ORDER BY total_revenue DESC
LIMIT 20
```

## Tips & Best Practices

### 1. Use Filters Wisely
- Always filter by date range to improve performance
- Use `attributed_station_id IS NULL` for network totals
- Use `attributed_station_id IS NOT NULL` to exclude network row

### 2. Aggregation Strategy
- Gold tables are already pre-aggregated daily/monthly
- For further aggregation (weekly, quarterly), use `DATE_TRUNC()`
- Avoid aggregating bronze tables directly (use gold instead)

### 3. Performance Optimization
- Create "Questions" and save them (cached)
- Avoid `SELECT *` on large tables
- Use date filters to limit data scanned
- Limit results with `LIMIT` clause

### 4. Data Freshness
- Gold tables refreshed when you run `dbt run`
- Check `_dbt_updated_at` column to see last refresh time
- Set up Dagster to automate daily refreshes

## Common Queries

### Network Total vs Station-Level

**Network Total** (1 row for entire network):
```sql
-- Network revenue
SELECT * FROM gold_daily_revenue
WHERE attributed_station_id IS NULL
```

**Station-Level** (1 row per station):
```sql
-- Individual station revenue
SELECT * FROM gold_daily_revenue
WHERE attributed_station_id IS NOT NULL
```

### Time Period Filters

**Last 7 days**:
```sql
WHERE report_date >= CURRENT_DATE - INTERVAL '7 days'
```

**Last 30 days**:
```sql
WHERE report_date >= CURRENT_DATE - INTERVAL '30 days'
```

**This month**:
```sql
WHERE report_date >= DATE_TRUNC('month', CURRENT_DATE)
```

**Last month**:
```sql
WHERE report_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
    AND report_date < DATE_TRUNC('month', CURRENT_DATE)
```

### Aggregation Patterns

**Daily → Weekly**:
```sql
SELECT
    DATE_TRUNC('week', report_date) as week,
    SUM(revenue) as weekly_revenue
FROM gold_daily_revenue
GROUP BY DATE_TRUNC('week', report_date)
```

**Daily → Monthly**:
```sql
SELECT
    DATE_TRUNC('month', report_date) as month,
    SUM(revenue) as monthly_revenue
FROM gold_daily_revenue
GROUP BY DATE_TRUNC('month', report_date)
```

## Troubleshooting

### Issue: Tables not appearing in Metabase

**Solution**:
1. Check Metabase is running: `docker ps | grep metabase`
2. Check database connection: Settings → Databases → Test connection
3. Sync schema: Settings → Databases → "Sync database schema now"
4. Restart if needed: `docker-compose restart metabase`

### Issue: Queries returning no data

**Check**:
1. Verify data exists: `jetx-lake gold revenue`
2. Check date filters (too restrictive?)
3. Check station filters (wrong station_id?)
4. Verify `lakehouse.duckdb` file is mounted: `docker exec jetx-metabase ls -lh /app/lakehouse.duckdb`

### Issue: Slow query performance

**Solutions**:
1. Add date range filter
2. Reduce number of rows with `LIMIT`
3. Use gold tables instead of bronze
4. Check if dbt models need optimization

### Issue: "Table not found" error

**Solutions**:
1. Check table name (case-sensitive in DuckDB)
2. Verify init SQL ran: Check Metabase logs
3. Test query in DuckDB directly: `duckdb transform/lakehouse.duckdb`

## PowerBI Integration

To connect PowerBI to this lakehouse:

### Option 1: DuckDB ODBC Driver (Recommended)

1. Install [DuckDB ODBC driver](https://github.com/duckdb/duckdb-odbc)
2. Configure ODBC data source pointing to `transform/lakehouse.duckdb`
3. In PowerBI: Get Data → ODBC → Select DuckDB source
4. Load tables from `main_gold` schema

### Option 2: Export to Parquet

```bash
# Export gold tables to Parquet
python -c "
import duckdb
conn = duckdb.connect('transform/lakehouse.duckdb')
conn.execute(\"COPY main_gold.gold_daily_revenue TO 'export/daily_revenue.parquet'\")
"
```

Then in PowerBI: Get Data → Parquet file

## Next Steps

1. **Build Dashboards**: Create executive, station, and subscription dashboards
2. **Set Up Alerts**: Configure email alerts for key metrics
3. **Automate Refresh**: Set up Dagster to refresh data daily
4. **User Training**: Train business users on Metabase
5. **Document Metrics**: Create data dictionary for all gold tables

## Resources

- **Metabase Docs**: https://www.metabase.com/docs/latest/
- **SQL Reference**: https://duckdb.org/docs/sql/introduction
- **Gold Layer Models**: `transform/models/gold/`
- **Architecture**: [docs/ARCHITECTURE.md](ARCHITECTURE.md)
