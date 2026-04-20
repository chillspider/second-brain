# PowerBI Connection to JetX Lakehouse

## Overview

Connect PowerBI Desktop (Windows) to your DuckDB lakehouse for live reporting and dashboards.

---

## Option 1: DuckDB ODBC Driver (Recommended) ⭐

This provides a **live connection** to your lakehouse. Data refreshes whenever you run `dbt run`.

### Prerequisites

- Windows machine with PowerBI Desktop
- WSL2 with JetX Lakehouse running
- Network access to WSL (localhost)

### Step 1: Install DuckDB ODBC Driver

1. **Download the driver:**
   - Go to: https://github.com/duckdb/duckdb-odbc/releases
   - Download latest Windows installer: `duckdb_odbc-windows-amd64.exe`
   - Example: `duckdb_odbc-1.2.0-windows-amd64.exe`

2. **Run the installer:**
   - Double-click the `.exe` file
   - Follow installation wizard
   - Default location: `C:\Program Files\DuckDB\ODBC Driver`

3. **Verify installation:**
   - Open "ODBC Data Sources (64-bit)" from Windows Start menu
   - Go to "Drivers" tab
   - You should see "DuckDB Driver" listed

### Step 2: Configure ODBC Data Source

1. **Open ODBC Data Source Administrator:**
   - Press `Win + R`
   - Type: `odbcad32.exe`
   - Click OK

2. **Add User DSN:**
   - Click "User DSN" tab
   - Click "Add" button
   - Select "DuckDB Driver"
   - Click "Finish"

3. **Configure Connection:**

   **Data Source Name (DSN):**
   ```
   JetX Lakehouse
   ```

   **Database:**
   ```
   \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\transform\lakehouse.duckdb
   ```

   **OR** (alternative path format):
   ```
   \\wsl$\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\transform\lakehouse.duckdb
   ```

   **Access Mode:**
   - ☑ Read Only

   **Configuration (Advanced):**
   - Leave other fields empty unless needed

4. **Test Connection:**
   - Click "Test" button
   - Should show: "Connection successful"
   - If error, verify WSL is running and path is correct

### Step 3: Connect PowerBI

1. **Open PowerBI Desktop**

2. **Get Data:**
   - Click "Home" → "Get Data"
   - Search for "ODBC"
   - Select "ODBC" → Click "Connect"

3. **Select Data Source:**
   - **Data source name (DSN)**: Select "JetX Lakehouse"
   - **Advanced options**: Leave empty
   - Click "OK"

4. **Navigator Window:**
   - You'll see available schemas:
     - `main_gold` ← **Use this for reports!**
     - `main_silver`
     - `main_staging`

5. **Load Gold Tables:**
   - Expand `main_gold`
   - Select tables you need:
     - ☑ `gold_daily_revenue`
     - ☑ `gold_station_performance`
     - ☑ `gold_subscription_mrr`
   - Click "Load" or "Transform Data"

6. **Data Refresh:**
   - In PowerBI: Home → Refresh
   - This re-queries DuckDB
   - Make sure you run `dbt run` first to update data!

---

## Option 2: Parquet File Export (Snapshot Data)

This creates **static snapshots** of your data as Parquet files.

### Step 1: Export Gold Tables to Parquet

On your WSL machine, create an export script:

```bash
# Create export directory
mkdir -p /home/jimmy/dev/jetx-lakehouse/export

# Create export script
cat > /home/jimmy/dev/jetx-lakehouse/export_to_parquet.py << 'EOF'
#!/usr/bin/env python3
"""
Export JetX Lakehouse gold tables to Parquet files for PowerBI.
"""
import duckdb
from pathlib import Path

# Connect to lakehouse
conn = duckdb.connect('transform/lakehouse.duckdb')

# Export directory
export_dir = Path('export')
export_dir.mkdir(exist_ok=True)

# Gold tables to export
gold_tables = [
    'gold_daily_revenue',
    'gold_station_performance',
    'gold_subscription_mrr',
]

print("Exporting gold tables to Parquet...")
for table in gold_tables:
    output_file = export_dir / f"{table}.parquet"

    query = f"""
        COPY main_gold.{table}
        TO '{output_file}' (FORMAT PARQUET, COMPRESSION ZSTD)
    """

    conn.execute(query)

    # Get stats
    count = conn.execute(f"SELECT COUNT(*) FROM main_gold.{table}").fetchone()[0]
    size = output_file.stat().st_size / (1024 * 1024)  # MB

    print(f"  ✓ {table}: {count:,} rows ({size:.1f} MB)")

print(f"\nExport complete! Files in: {export_dir.absolute()}")
print("\nCopy these files to Windows and import into PowerBI:")
print("  PowerBI → Get Data → Parquet → Select file")
EOF

chmod +x export_to_parquet.py
```

### Step 2: Run Export

```bash
cd /home/jimmy/dev/jetx-lakehouse
python3 export_to_parquet.py
```

Output:
```
Exporting gold tables to Parquet...
  ✓ gold_daily_revenue: 3,653 rows (0.8 MB)
  ✓ gold_station_performance: 7,956 rows (1.2 MB)
  ✓ gold_subscription_mrr: 181 rows (0.1 MB)

Export complete! Files in: /home/jimmy/dev/jetx-lakehouse/export
```

### Step 3: Copy Files to Windows

**Option A: Windows Explorer**
```
1. Open Windows Explorer
2. Navigate to: \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export
3. Copy .parquet files to Windows (e.g., C:\JetX\data\)
```

**Option B: PowerShell**
```powershell
# From PowerShell
Copy-Item \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\export\*.parquet C:\JetX\data\
```

### Step 4: Import to PowerBI

1. **Open PowerBI Desktop**

2. **Get Data:**
   - Click "Home" → "Get Data"
   - Search for "Parquet"
   - Select "Parquet" → Click "Connect"

3. **Select Files:**
   - Navigate to `C:\JetX\data\`
   - Select all `.parquet` files
   - Click "Open"

4. **Load Data:**
   - PowerBI will show preview
   - Click "Load"

5. **Refresh Process:**
   - Data is static (snapshot)
   - To refresh: Re-run export script, copy files, refresh in PowerBI

---

## Option 3: HTTP/REST API (Advanced)

Create a REST API endpoint that PowerBI can query.

### Setup DuckDB HTTP Server

```bash
# Install DuckDB HTTP extension
pip install duckdb-http

# Create API server (api_server.py)
cat > /home/jimmy/dev/jetx-lakehouse/api_server.py << 'EOF'
from flask import Flask, jsonify, request
import duckdb

app = Flask(__name__)

@app.route('/api/gold/<table>', methods=['GET'])
def get_gold_table(table):
    """Return gold table as JSON"""
    allowed_tables = ['gold_daily_revenue', 'gold_station_performance', 'gold_subscription_mrr']

    if table not in allowed_tables:
        return jsonify({'error': 'Table not found'}), 404

    conn = duckdb.connect('transform/lakehouse.duckdb')

    # Optional filters from query params
    where = request.args.get('where', '')
    limit = request.args.get('limit', '1000')

    query = f"SELECT * FROM main_gold.{table}"
    if where:
        query += f" WHERE {where}"
    query += f" LIMIT {limit}"

    result = conn.execute(query).df().to_dict('records')
    return jsonify(result)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Run server
python3 api_server.py
```

Then in PowerBI: Get Data → Web → `http://localhost:5000/api/gold/gold_daily_revenue`

---

## Comparison: Which Option to Choose?

| Feature | ODBC (Option 1) | Parquet (Option 2) | REST API (Option 3) |
|---------|----------------|-------------------|-------------------|
| **Setup Time** | 10 minutes | 5 minutes | 30 minutes |
| **Real-time** | Yes (live query) | No (snapshot) | Yes (live query) |
| **Performance** | Fast (direct) | Fastest (cached) | Medium (network) |
| **Data Volume** | Unlimited | Limited by disk | Limited by memory |
| **Refresh** | Click refresh | Re-export + copy | Click refresh |
| **Best For** | Production dashboards | One-time reports | Multi-user access |

**Recommendation:**
- **Production use** → Option 1 (ODBC)
- **Quick reports** → Option 2 (Parquet)
- **Shared dashboards** → Option 3 (REST API)

---

## Troubleshooting

### ODBC: "Cannot find database file"

**Problem**: PowerBI can't access WSL file path

**Solutions:**

1. **Verify WSL is running:**
   ```bash
   wsl -l -v
   # Should show Ubuntu-24.04 as "Running"
   ```

2. **Try alternative path format:**
   ```
   \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\transform\lakehouse.duckdb
   ```

3. **Check file exists:**
   ```powershell
   # From PowerShell
   Test-Path \\wsl.localhost\Ubuntu-24.04\home\jimmy\dev\jetx-lakehouse\transform\lakehouse.duckdb
   # Should return: True
   ```

4. **Copy file to Windows:**
   ```bash
   # Alternative: Copy DuckDB file to Windows
   cp transform/lakehouse.duckdb /mnt/c/JetX/lakehouse.duckdb

   # Then use in ODBC:
   # C:\JetX\lakehouse.duckdb
   ```

### ODBC: "Driver not found"

**Problem**: ODBC driver not installed correctly

**Solutions:**

1. **Verify driver installation:**
   - Open "ODBC Data Sources (64-bit)"
   - Check "Drivers" tab for "DuckDB Driver"

2. **Reinstall driver:**
   - Download again from GitHub releases
   - Run as Administrator

3. **Use 64-bit ODBC manager:**
   - NOT: `C:\Windows\SysWOW64\odbcad32.exe` (32-bit)
   - YES: `C:\Windows\System32\odbcad32.exe` (64-bit)

### Parquet: "File format not recognized"

**Problem**: PowerBI doesn't support Parquet natively (older versions)

**Solutions:**

1. **Update PowerBI Desktop:**
   - Download latest from Microsoft

2. **Export as CSV instead:**
   ```python
   conn.execute(f"COPY main_gold.{table} TO '{output_file}.csv' (HEADER, DELIMITER ',')")
   ```

3. **Use XLSX format:**
   ```python
   df = conn.execute(f"SELECT * FROM main_gold.{table}").df()
   df.to_excel(f'export/{table}.xlsx', index=False)
   ```

### Performance: Slow queries in PowerBI

**Solutions:**

1. **Limit date range:**
   - Add date filters in PowerBI
   - Only load recent data (last 90 days)

2. **Use DirectQuery mode:**
   - In ODBC setup, select "DirectQuery" instead of "Import"
   - Queries run on-demand (slower but fresh data)

3. **Pre-aggregate in dbt:**
   - Create more specific gold tables
   - Example: `gold_monthly_summary` instead of daily

---

## Best Practices

### 1. Data Refresh Schedule

**For ODBC (Live Connection):**
```bash
# Automate with cron (daily 6 AM)
0 6 * * * cd /home/jimmy/dev/jetx-lakehouse && dbt run --select gold
```

Then in PowerBI:
- Set refresh schedule: Daily at 7 AM
- Gives 1 hour buffer for dbt to complete

**For Parquet (Snapshot):**
```bash
# Export + copy script
0 6 * * * cd /home/jimmy/dev/jetx-lakehouse && python3 export_to_parquet.py && cp export/*.parquet /mnt/c/JetX/data/
```

### 2. Security

**ODBC:**
- Use read-only ODBC connection (already configured)
- DuckDB file has proper permissions: `chmod 644 lakehouse.duckdb`

**Parquet:**
- Exported files are static (no security needed)
- Can be shared freely

**REST API:**
- Add authentication (API key, OAuth)
- Use HTTPS in production
- Rate limit to prevent abuse

### 3. Performance Optimization

**PowerBI Settings:**
- **Data Load**: Use Import mode for small datasets (<1M rows)
- **DirectQuery**: Use for large datasets (real-time queries)
- **Incremental Refresh**: Load only new data (last 30 days)

**DuckDB Optimization:**
```sql
-- In PowerBI "Transform Data", add date filter
let
    Source = Odbc.DataSource("dsn=JetX Lakehouse"),
    gold_daily_revenue = Source{[Schema="main_gold",Item="gold_daily_revenue"]}[Data],
    FilteredRows = Table.SelectRows(gold_daily_revenue, each [report_date] >= Date.AddDays(Date.From(DateTime.LocalNow()), -90))
in
    FilteredRows
```

---

## Example PowerBI Reports

### 1. Daily Revenue Dashboard

**Data Sources:**
- `gold_daily_revenue`

**Visuals:**
- Line chart: Revenue trend (last 30 days)
- Card: Total revenue this month
- Table: Top 10 stations by revenue
- Map: Revenue by zone

**Measures:**
```dax
Total Revenue = SUM(gold_daily_revenue[revenue])
Revenue Growth % =
    VAR CurrentMonth = [Total Revenue]
    VAR PreviousMonth = CALCULATE([Total Revenue], DATEADD(gold_daily_revenue[report_date], -1, MONTH))
    RETURN DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth)
```

### 2. Station Performance Scorecard

**Data Sources:**
- `gold_station_performance`
- `dim_stations` (from silver)

**Visuals:**
- Matrix: Station performance metrics
- Gauge: A30/S30 vs target
- Scatter plot: Revenue vs wash count
- Slicer: Date range, zone

**Measures:**
```dax
Latest A30 =
    CALCULATE(
        MAX(gold_station_performance[a30]),
        FILTER(ALL(gold_station_performance), gold_station_performance[report_date] = MAX(gold_station_performance[report_date]))
    )
```

### 3. Subscription MRR Report

**Data Sources:**
- `gold_subscription_mrr`

**Visuals:**
- Area chart: MRR growth over time
- KPI: Current MRR vs target
- Donut chart: Silver vs Gold split
- Line chart: Churn trend

**Measures:**
```dax
MRR Growth = [Current MRR] - [Previous Month MRR]
Churn Rate % = DIVIDE([churned_this_month], [active_subscriptions])
```

---

## Next Steps

1. **Install ODBC driver** on your Windows machine
2. **Configure DSN** pointing to lakehouse.duckdb
3. **Connect PowerBI** and load gold tables
4. **Build your first dashboard** (revenue trend)
5. **Set up refresh schedule** (daily morning refresh)

---

## Resources

- **DuckDB ODBC**: https://github.com/duckdb/duckdb-odbc
- **PowerBI Docs**: https://learn.microsoft.com/en-us/power-bi/
- **Gold Layer Models**: `transform/models/gold/`
- **Architecture**: [ARCHITECTURE.md](ARCHITECTURE.md)
