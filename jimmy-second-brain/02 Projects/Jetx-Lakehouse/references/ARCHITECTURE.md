# JetX Lakehouse Architecture

## Overview

The JetX Lakehouse is a modern data platform using the **Medallion Architecture** (Bronze → Silver → Gold) with MinIO (S3-compatible storage), DuckDB (analytics database), and Apache Iceberg (table format).

## Data Storage Layers

### Bronze Layer (Raw Data)
- **Storage**: MinIO S3 (`s3://lakehouse/bronze_*/`)
- **Format**: Apache Iceberg tables (Parquet files + metadata)
- **Schema**: Multiple domains (operations, subscriptions, vouchers, customers, vehicles, reference)
- **Written by**: Ingestion pipeline (`jetx-lake ingest`)
- **Read by**: dbt staging models
- **Persistence**: Persistent (never deleted, append-only)

**Example structure:**
```
s3://lakehouse/
├── bronze_operations/
│   ├── orders/
│   │   ├── metadata/
│   │   └── data/*.parquet
│   ├── stations/
│   └── devices/
├── bronze_subscriptions/
│   ├── subscriptions/
│   └── subscription_payments/
└── bronze_customers/
    └── users/
```

### Silver Layer (Cleaned, Conformed Data)
- **Storage**: DuckDB file (`transform/lakehouse.duckdb`)
- **Schema**: `main_silver`
- **Format**: DuckDB native tables
- **Written by**: dbt models (`jetx_lakehouse.silver.*`)
- **Read by**: Gold models, CLI commands
- **Materialization**: Tables (recreated on every `dbt run`)
- **Persistence**: Persistent, but recreated from bronze on each run

**Table types:**
- **Dimensions**: `dim_stations`, `dim_customers`, `dim_vehicles`, etc.
- **Facts**: `fct_transactions`, `fct_washes`, `fct_subscriptions`, `fct_daily_active_users`, `fct_daily_churn`
- **Reference**: `ref_geo_vietnam`

### Gold Layer (Business Metrics)
- **Storage**: DuckDB file (`transform/lakehouse.duckdb`)
- **Schema**: `main_gold`
- **Format**: DuckDB native tables
- **Written by**: dbt models (`jetx_lakehouse.gold.*`)
- **Read by**: Metabase, PowerBI, CLI commands
- **Materialization**: Tables (recreated on every `dbt run --select gold`)
- **Persistence**: Persistent, but recreated from silver on each run

**Gold tables:**
- `gold_daily_revenue`: Daily revenue metrics by station (1 row = 1 day + 1 station)
- `gold_station_performance`: Daily station performance (59 columns: A30/A90/S30/S90, revenue, churn, etc.)
- `gold_subscription_mrr`: Monthly recurring revenue by station (1 row = 1 month + 1 station)

## Data Flow

```
┌─────────────────┐
│ Source DB       │
│ (PostgreSQL)    │
└────────┬────────┘
         │ Ingestion Pipeline
         ▼
┌─────────────────┐
│ Bronze Layer    │
│ (MinIO S3)      │ ← Persistent, Iceberg format
│ Parquet files   │
└────────┬────────┘
         │ dbt staging models
         ▼
┌─────────────────┐
│ Silver Layer    │
│ (DuckDB)        │ ← Recreated on dbt run
│ Cleaned tables  │
└────────┬────────┘
         │ dbt gold models
         ▼
┌─────────────────┐
│ Gold Layer      │
│ (DuckDB)        │ ← Recreated on dbt run
│ Business metrics│
└────────┬────────┘
         │
         ├─► Metabase (dashboards)
         ├─► PowerBI (analytics)
         └─► CLI (ad-hoc queries)
```

## Component Details

### 1. Ingestion Pipeline
- **Tool**: Custom Python (`ingestion/`)
- **Command**: `jetx-lake ingest table <domain>.<table>`
- **Process**:
  1. Extract from PostgreSQL
  2. Load to MinIO S3 as Iceberg/Parquet
  3. Update Iceberg metadata catalog
- **Configuration**: Environment variables (`.env`)

### 2. dbt Transformations
- **Tool**: dbt-duckdb
- **Location**: `transform/`
- **Command**: `cd transform && dbt run`
- **Process**:
  1. Staging models read from bronze (S3)
  2. Create/replace silver tables in DuckDB
  3. Create/replace gold tables in DuckDB
- **Configuration**: `transform/profiles.yml`

### 3. Metabase (Visualization)
- **Type**: BI tool with DuckDB driver
- **Access**: http://localhost:3001
- **Database**: Hybrid approach
  - **In-memory DuckDB** with init SQL that:
    - Attaches `lakehouse.duckdb` (read-only)
    - Creates views for bronze layer (S3)
    - Creates views for gold layer (from attached DB)
- **Configuration**: `docker/metabase/duckdb_init.sql`

### 4. CLI Tool
- **Command**: `jetx-lake gold <command>`
- **Database**: Direct connection to `transform/lakehouse.duckdb`
- **Features**:
  - `jetx-lake gold revenue`: Daily revenue report
  - `jetx-lake gold mrr`: MRR subscription metrics
  - `jetx-lake gold performance`: Station performance rankings

## File Locations

```
jetx-lakehouse/
├── ingestion/              # Bronze layer pipeline
│   ├── sources/            # Database connectors
│   ├── extractors/         # Data extraction logic
│   └── loaders/            # Iceberg writers
├── transform/              # Silver + Gold (dbt)
│   ├── models/
│   │   ├── staging/        # Bronze → Silver
│   │   ├── silver/         # Dimension + fact tables
│   │   └── gold/           # Business metrics
│   ├── lakehouse.duckdb    # ⭐ Main analytics database
│   └── profiles.yml        # dbt DuckDB config
├── cli/                    # Command-line interface
│   └── commands/gold.py    # Gold layer CLI
├── docker/                 # Container configs
│   └── metabase/
│       └── duckdb_init.sql # Metabase initialization
└── docs/                   # Documentation
    └── ARCHITECTURE.md     # This file
```

## Key Configuration Files

### 1. dbt Profile (`transform/profiles.yml`)
```yaml
jetx_lakehouse:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: 'lakehouse.duckdb'  # Persistent DuckDB file
      extensions:
        - httpfs
      settings:
        s3_endpoint: 'localhost:9000'
        s3_access_key_id: 'minioadmin'
        s3_secret_access_key: 'minioadmin'
        s3_use_ssl: false
```

### 2. dbt Project (`transform/dbt_project.yml`)
```yaml
models:
  jetx_lakehouse:
    staging:
      +materialized: view
      +schema: staging
    silver:
      +materialized: table  # ← Recreated on dbt run
      +schema: silver
    gold:
      +materialized: table  # ← Recreated on dbt run
      +schema: gold
```

### 3. Metabase Init SQL (`docker/metabase/duckdb_init.sql`)
```sql
-- Attach persistent DuckDB file
ATTACH '/app/lakehouse.duckdb' AS lakehouse (READ_ONLY);

-- Create views for bronze (S3)
CREATE OR REPLACE VIEW bronze_orders AS
SELECT * FROM read_parquet('s3://lakehouse/bronze_operations/orders/data/*.parquet');

-- Create views for gold (from attached DB)
CREATE OR REPLACE VIEW gold_daily_revenue AS
SELECT * FROM lakehouse.main_gold.gold_daily_revenue;
```

## Data Refresh Workflow

### Daily Refresh (Automated via Dagster)
```bash
# 1. Ingest new data to bronze
jetx-lake ingest table operations.orders
jetx-lake ingest table subscriptions.subscriptions

# 2. Rebuild silver + gold
cd transform
dbt run

# 3. Metabase picks up changes automatically
# (views read from attached lakehouse.duckdb)
```

### Manual Refresh
```bash
# Full refresh
jetx-lake ingest domain operations
cd transform && dbt run

# Gold only
cd transform && dbt run --select gold

# Specific table
cd transform && dbt run --select gold_daily_revenue
```

## Querying the Lakehouse

### 1. Via Metabase (UI)
- Access: http://localhost:3001
- Database: "JetX-LakeHouse-DuckDB"
- Tables: `gold_daily_revenue`, `gold_station_performance`, `gold_subscription_mrr`

### 2. Via CLI
```bash
# Revenue report
jetx-lake gold revenue --range last-30-day

# Station performance
jetx-lake gold performance --sort revenue --limit 10

# MRR metrics
jetx-lake gold mrr --months 12
```

### 3. Via DuckDB CLI
```bash
duckdb transform/lakehouse.duckdb

# Query gold tables
SELECT * FROM main_gold.gold_daily_revenue WHERE report_date >= '2026-01-01';

# Query silver tables
SELECT station_name, zone_name FROM main_silver.dim_stations;
```

### 4. Via Python
```python
import duckdb

conn = duckdb.connect('transform/lakehouse.duckdb')
df = conn.execute("""
    SELECT report_date, SUM(revenue) as total_revenue
    FROM main_gold.gold_daily_revenue
    WHERE report_date BETWEEN '2026-01-01' AND '2026-01-31'
    GROUP BY report_date
    ORDER BY report_date
""").df()
```

## Performance Characteristics

### Bronze Layer (S3/Iceberg)
- **Read**: ~100-500 MB/s (depends on network)
- **Write**: ~50-200 MB/s
- **Storage**: Columnar Parquet (10:1 compression ratio)
- **Use case**: Archival, ad-hoc exploration

### Silver Layer (DuckDB)
- **Read**: ~2-5 GB/s (memory-mapped files)
- **Write**: ~500 MB/s
- **Storage**: DuckDB native (optimized for analytics)
- **Use case**: Data modeling, transformations

### Gold Layer (DuckDB)
- **Read**: ~5-10 GB/s (small, pre-aggregated)
- **Query**: Sub-second for most dashboards
- **Storage**: Highly compressed (aggregated data)
- **Use case**: Dashboards, reports

## Security & Access Control

### Current State (Development)
- **MinIO**: Default credentials (`minioadmin`/`minioadmin`)
- **DuckDB**: File-based, no authentication
- **Metabase**: User authentication via web UI
- **PostgreSQL sources**: Credentials in `.env`

### Production Recommendations
1. **MinIO**: Use IAM policies, rotate credentials
2. **DuckDB**: File permissions (chmod 600), read-only mounts for Metabase
3. **Metabase**: SSO integration, role-based access
4. **Secrets**: Use secret management (HashiCorp Vault, AWS Secrets Manager)

## Backup & Recovery

### Bronze Layer
- **Backup**: MinIO supports versioning, replication
- **Recovery**: Restore from MinIO backup or re-ingest from source

### Silver + Gold Layers
- **Backup**: `lakehouse.duckdb` file backups (daily snapshot)
- **Recovery**: Run `dbt run` to rebuild from bronze

### Disaster Recovery
```bash
# Worst case: rebuild everything from source databases
jetx-lake ingest domain operations
jetx-lake ingest domain subscriptions
cd transform && dbt run
```

## Monitoring & Observability

### Data Quality
- **dbt tests**: Schema tests, uniqueness, not null
- **Custom tests**: Business logic validation

### Performance
- **dbt**: Query timing in logs
- **Metabase**: Query performance tracking
- **MinIO**: Prometheus metrics endpoint

### Orchestration
- **Dagster**: Job monitoring, alerts, retry logic

## Cost Breakdown (Estimated)

For 1M records/day, 100GB data:
- **MinIO storage**: ~$0.02/GB/month (S3-compatible)
- **DuckDB**: Free (embedded, no server costs)
- **Metabase**: Free (community edition)
- **Compute**: AWS EC2 t3.medium (~$30/month)

**Total**: ~$32/month for entire stack

## Comparison to Alternatives

| Feature | Current (DuckDB + MinIO) | Snowflake | BigQuery |
|---------|--------------------------|-----------|----------|
| Storage cost | $0.02/GB | $23/TB | $20/TB |
| Compute cost | Free (local) | $2/credit | $5/TB scanned |
| Setup time | 1 hour | 1 day | 1 day |
| Latency | <100ms | ~500ms | ~1000ms |
| Scalability | Single node | Unlimited | Unlimited |

## Future Enhancements

1. **Delta Lake Migration**: Replace Iceberg with Delta Lake for ACID
2. **Incremental Models**: Use dbt incremental for large tables
3. **Partitioning**: Partition bronze tables by date
4. **Caching**: Add Redis for frequently accessed gold metrics
5. **Real-time**: Add Kafka for streaming ingestion
6. **Multi-region**: Replicate MinIO across regions

## Troubleshooting

### Issue: Metabase not showing gold tables
**Solution**: Restart Metabase after dbt run
```bash
docker-compose restart metabase
```

### Issue: dbt can't read bronze tables
**Solution**: Check S3 connection settings in `profiles.yml`
```bash
# Test S3 access
python -c "
import duckdb
conn = duckdb.connect()
conn.execute(\"INSTALL httpfs; LOAD httpfs\")
conn.execute(\"SET s3_endpoint='localhost:9000'\")
conn.execute(\"SELECT COUNT(*) FROM read_parquet('s3://lakehouse/bronze_operations/orders/data/*.parquet')\")
"
```

### Issue: lakehouse.duckdb file is huge
**Solution**: Run VACUUM to reclaim space
```sql
VACUUM;
CHECKPOINT;
```

## Resources

- **DuckDB Docs**: https://duckdb.org/docs/
- **dbt Docs**: https://docs.getdbt.com/
- **Apache Iceberg**: https://iceberg.apache.org/
- **MinIO**: https://min.io/docs/minio/
- **Metabase**: https://www.metabase.com/docs/

## Support

For questions or issues:
1. Check this documentation
2. Review session logs in `docs/SESSION_LOG_*.md`
3. Check dbt logs: `transform/logs/dbt.log`
4. Review Dagster UI: http://localhost:3002
