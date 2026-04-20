# Decision Log

This document tracks major architectural and implementation decisions for the JetX Lakehouse project.

---

## 2026-01-15: Custom Dashboard over Metabase

**Decision**: Build custom Refine + Ant Design dashboard instead of relying on Metabase

**Context**:
- Metabase is quirky with SQL editor
- DuckDB locking conflicts when Metabase holds connection
- Team is technical and comfortable with SQL
- Already have well-structured SQL queries in `/queries` folder

**Approach**:
1. **SQL-First Development**: Write and test SQL queries in `/queries` folder
2. **Organized Structure**: Separate by layer (bronze/silver/gold) and query type
3. **CLI Integration**: Each validated query gets a `jetx-lake` CLI command
4. **Future Custom Frontend**: Build Refine + FastAPI + DuckDB read-only dashboard

**Keep Metabase For**:
- Ad-hoc exploration by non-technical users
- Quick prototyping of new visualizations
- Secondary/backup BI tool

**Immediate Action Items**:
- ✅ Organize `/queries` folder with subfolders
- ✅ Document query structure and naming conventions
- Create CLI commands for validated queries
- Build custom dashboard (future phase)

---

## Query Organization Structure

```
queries/
├── gold/
│   ├── revenue/           # Revenue analysis queries
│   ├── performance/       # Station performance queries
│   ├── customers/         # Customer behavior queries
│   └── powerbi/          # PowerBI-optimized exports
├── silver/
│   ├── dimensional/       # Dimension table queries
│   └── factual/          # Fact table queries
├── bronze/
│   └── raw/              # Raw data inspection queries
└── README.md             # Query catalog and usage guide
```

**Naming Convention**: `{grain}_{metric}_{aggregation}.sql`
- Example: `daily_revenue_by_station.sql`
- Example: `monthly_churn_trend.sql`
- Example: `all_time_station_summary.sql`

---

## Previous Decisions

### 2026-01-14: Gold Layer Metrics Framework
- Established daily grain as foundation
- Pre-aggregated A30/A90/S30/S90 active users
- Separate revenue and performance tables

### 2026-01-13: Medallion Architecture
- Bronze: Raw ingestion to S3/Iceberg
- Silver: Cleaned dimensions and facts in DuckDB
- Gold: Business metrics in DuckDB

### 2026-01-12: DuckDB over Data Warehouse
- Local-first development
- Single file database for transforms
- dbt for transformation logic

---

**Last Updated**: 2026-01-15
