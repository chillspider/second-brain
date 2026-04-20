# Export Automation Guide

This document explains how Parquet exports are automated in the JetX Lakehouse.

---

## Current Setup

### Manual Export Scripts

Three standalone Python scripts for on-demand exports:

1. **export_for_powerbi.py** - Gold layer aggregates (56 KB)
   - `station_summary_all_time.parquet`
   - `station_summary_monthly.parquet`
   - `daily_revenue.parquet`

2. **export_silver_bronze.py** - Silver/Bronze raw data (37 MB)
   - 14 silver dimension/fact tables
   - 4 bronze/staging tables

### Automated Export (dbt Workflow)

**Script**: `dbt_run_and_export.sh`

**What it does**:
1. Stops Metabase (releases DuckDB lock)
2. Runs dbt transformations
3. **Automatically exports all layers to Parquet**
4. Restarts Metabase

**Usage**:
```bash
./dbt_run_and_export.sh
```

**When it runs**: Only when you explicitly execute this script

---

## Automation Options

### Option 1: Run After Every dbt Build (Current)

✅ **Already configured** in `dbt_run_and_export.sh`

Every time you run this script, it will:
- Build gold/silver models
- Export all layers to Parquet
- Files are ready for PowerBI immediately

**Pros**:
- Always have fresh exports
- No separate export step needed
- PowerBI files stay in sync with database

**Cons**:
- Adds ~10 seconds to dbt run time
- Exports even if you don't need them

---

### Option 2: Scheduled Cron Job

Export on a schedule (e.g., daily at 6 AM):

```bash
# Edit crontab
crontab -e

# Add this line (run daily at 6 AM)
0 6 * * * cd /home/jimmy/dev/jetx-lakehouse && ./dbt_run_and_export.sh >> logs/export.log 2>&1
```

**Pros**:
- Automatic daily refresh
- PowerBI users always have fresh data
- No manual intervention

**Cons**:
- Need to ensure dbt models are up to date
- Requires cron setup

---

### Option 3: Git Pre-Commit Hook

Export before every git commit (for data version control):

```bash
# Create .git/hooks/pre-commit
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
cd /home/jimmy/dev/jetx-lakehouse
echo "Exporting Parquet files..."
python3 export_for_powerbi.py
python3 export_silver_bronze.py
git add export/
EOF

chmod +x .git/hooks/pre-commit
```

**Pros**:
- Exports versioned with code
- Can track data changes over time

**Cons**:
- Slower commits
- Large binary files in git (not recommended)

---

### Option 4: Dagster Schedule (Production)

When you move to Dagster orchestration:

```python
# dagster_pipeline.py
from dagster import schedule, job, op

@op
def export_parquet():
    """Export all layers to Parquet."""
    subprocess.run(['python3', 'export_for_powerbi.py'])
    subprocess.run(['python3', 'export_silver_bronze.py'])

@job
def export_job():
    export_parquet()

@schedule(cron_schedule="0 6 * * *", job=export_job)
def daily_export_schedule():
    """Export Parquet files daily at 6 AM."""
    return {}
```

**Pros**:
- Integrated with orchestration
- Monitoring and alerting
- Production-ready

**Cons**:
- Requires Dagster setup

---

## Recommended Approach

### For Development (Now)

Use **Option 1** (already configured):
```bash
# Run dbt and auto-export
./dbt_run_and_export.sh
```

### For Production (Later)

Use **Option 2** (cron) or **Option 4** (Dagster):
- Automated daily exports
- PowerBI reports refresh automatically
- No manual intervention

---

## Export File Locations

```
export/
├── powerbi/          # Gold layer aggregates (56 KB) - PowerBI optimized
│   ├── station_summary_all_time.parquet    # 18 stations, all-time totals
│   ├── station_summary_monthly.parquet     # 147 rows, monthly breakdown
│   └── daily_revenue.parquet               # 3,635 rows, daily detail
├── silver/           # Silver layer cleaned data (28 MB)
│   ├── dim_customers.parquet               # 174K customers
│   ├── dim_stations.parquet                # 17 stations
│   ├── fct_washes.parquet                  # 108K washes
│   └── ... (14 files total)
└── bronze/           # Bronze/Staging layer raw data (9 MB)
    ├── stg_order_attribution.parquet       # 163K rows
    ├── stg_wallet_transactions.parquet     # 50K rows
    └── ... (4 files total)
```

**Important Naming**:
- The folder is called `powerbi/`, not `gold/`
- This clearly indicates these are PowerBI-optimized exports of gold layer data
- `gold/` folder (if it exists) is from old scripts and can be deleted

**Windows Access Paths**:
```
PowerBI:  \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export\powerbi
Silver:   \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export\silver
Bronze:   \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export\bronze
```

---

## Manual Export Commands

If you need to export without running dbt:

```bash
# Stop Metabase first (releases lock)
docker-compose stop metabase

# Export gold layer only (fast)
python3 export_for_powerbi.py

# Export silver/bronze (slower)
python3 export_silver_bronze.py

# Restart Metabase
docker-compose start metabase
```

---

## PowerBI Refresh Strategy

### Option A: Manual Refresh (Current)
1. Open PowerBI Desktop
2. Click **Refresh** button
3. PowerBI reloads from Parquet files

### Option B: Scheduled Refresh (PowerBI Service)
1. Publish report to PowerBI Service
2. Set up scheduled refresh (daily/hourly)
3. PowerBI Service reads from WSL path (if accessible)

### Option C: Power Automate + OneDrive
1. Use Power Automate to copy files to OneDrive
2. PowerBI Service refreshes from OneDrive
3. Works around WSL path access issues

---

## Troubleshooting

### Exports fail with "Database is locked"
```bash
# Stop Metabase first
docker-compose stop metabase

# Run export
./dbt_run_and_export.sh

# Metabase will auto-restart
```

### PowerBI can't access WSL path
**Solution 1**: Copy files to Windows
```bash
# From WSL
cp -r export /mnt/c/JetX/data/
```

**Solution 2**: Use UNC path in PowerBI
```
\\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export
```

### Exports are slow
- Gold layer export: ~2 seconds
- Silver layer export: ~8 seconds
- Bronze layer export: ~3 seconds
- Total: ~13 seconds

**Optimization**: Export only what you need
```bash
# Gold only (fastest)
python3 export_for_powerbi.py
```

---

## Next Steps

1. ✅ Use `./dbt_run_and_export.sh` for development
2. ⏳ Set up cron job for automated daily exports (Option 2)
3. ⏳ Move to Dagster orchestration for production (Option 4)
4. ⏳ Implement PowerBI Service scheduled refresh

---

**Last Updated**: 2026-01-15
