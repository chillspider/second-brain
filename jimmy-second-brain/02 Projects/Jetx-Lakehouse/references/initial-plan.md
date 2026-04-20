 JetX Lakehouse Implementation Plan

 Executive Summary

 Build a modern data lakehouse to replace the current PostgreSQL-based warehouse, designed to scale from current 120K washes
  to 1M+ washes while maintaining cost efficiency.

 ---
 Architecture Decision

 ┌─────────────────────────────────────────────────────────────────┐
 │                     JETX LAKEHOUSE                              │
 └─────────────────────────────────────────────────────────────────┘

 Source (PostgreSQL wash24h_prod)
     │
     │  Query-based ingestion
     ▼
 Landing Zone (S3/MinIO)
     │  landing/{source}/{table}/{date}/data.parquet
     ▼
 Bronze Layer (Iceberg tables - raw)
     │  bronze/{source}/{table}/
     ▼
 Silver Layer (Iceberg via dbt - cleaned)
     │  silver/core/     ← Shared dimensions
     │  silver/wash/     ← Wash service facts
     │  silver/{service}/ ← Future services
     ▼
 Gold Layer (Iceberg via dbt - business-ready)
     │  gold/analytics/
     │  gold/datamarts/
     ▼
 Serving (DuckDB queries → BI/API/Apps)

 Technology Stack

 | Component     | Technology              | Rationale                                   |
 |---------------|-------------------------|---------------------------------------------|
 | Storage       | MinIO (dev) / S3 (prod) | Object storage, infinite scale              |
 | Table Format  | Apache Iceberg          | Time-travel, schema evolution, open format  |
 | Compute       | DuckDB                  | Fast, in-memory, 96GB RAM = 5+ years runway |
 | Transforms    | dbt-duckdb              | SQL-based, version controlled, testable     |
 | Orchestration | Dagster                 | Python-native, asset-based, modern          |
 | Catalog       | PyIceberg (filesystem)  | Simple, no external catalog needed          |
 | Analytics/BI  | Metabase                | Open-source, self-hosted, SQL-native        |

 ---
 Data Sources

 Source Database (wash24h_prod)

 - 63 tables, ~2.5GB total
 - Key tables: orders (154K), order_items (119K), users (44K), subscriptions (2K)
 - 55 JSONB columns with rich nested data to normalize

 Current Warehouse (jetx_analytics_warehouse)

 - 57 tables, ~500MB total
 - Already has: dim_*, fact_*, analytics_*, datamart_*
 - Missing from source: device_logs (647K), car_detectors (70K), vouchers (32K), notifications (240K)

 ---
 Implementation Phases

 Phase 0: Foundation (This Sprint)

 Goal: Prove the stack works end-to-end with one table

 - Set up project structure
 - Configure MinIO (local development)
 - Set up PyIceberg + DuckDB integration
 - Ingest one table (dim_stations) from source → landing → bronze
 - Query bronze via DuckDB
 - Create one dbt model (silver)

 Deliverable: Can query SELECT * FROM silver.dim_stations via DuckDB

 Phase 1: Bronze Layer (Full Ingestion)

 Goal: Mirror all source tables to Iceberg

 Tables to ingest from source (wash24h_prod):

 Core (Priority 1):
 - orders → bronze.wash24h.orders
 - order_items → bronze.wash24h.order_items
 - users → bronze.wash24h.users
 - stations → bronze.wash24h.stations
 - devices → bronze.wash24h.devices
 - subscriptions → bronze.wash24h.subscriptions
 - subscription_payments → bronze.wash24h.subscription_payments
 - vehicles → bronze.wash24h.vehicles
 - membership_plans → bronze.wash24h.membership_plans
 - modes → bronze.wash24h.modes

 Extended (Priority 2):
 - vouchers → bronze.wash24h.vouchers
 - packages → bronze.wash24h.packages
 - car_detectors → bronze.wash24h.car_detectors
 - device_logs → bronze.wash24h.device_logs (647K rows!)
 - referrals → bronze.wash24h.referrals

 Supporting:
 - order_transactions → bronze.wash24h.order_transactions
 - subscription_logs → bronze.wash24h.subscription_logs
 - invoices → bronze.wash24h.invoices
 - invoice_items → bronze.wash24h.invoice_items

 Partitioning Strategy:
 - orders, order_items: partition by created_at (monthly)
 - device_logs: partition by created_at (monthly)
 - Others: no partition (small tables)

 Incremental Strategy:
 - Use updated_at or created_at for incremental loads
 - Full refresh for dimension tables (small)

 Phase 2: Silver Layer (dbt Models - Dimensions & Facts)

 Goal: Clean, deduplicated, normalized tables

 Core Dimensions (silver/core/):
 - dim_customers (unified from users)
 - dim_stations (from stations + station_locations)
 - dim_devices (from devices)
 - dim_vehicles (from vehicles + car_detectors)
 - dim_membership_plans (from membership_plans)
 - dim_wash_modes (from modes)
 - dim_vouchers (from vouchers) ← NEW
 - dim_packages (from packages)
 - dim_payment_methods (extracted from JSONB) ← NEW

 Wash Facts (silver/wash/):
 - fact_washes (from orders + order_items, JSONB flattened)
 - fact_transactions (from order_transactions)
 - fact_subscription_events (from subscription_logs)
 - fact_vehicle_detections (from car_detectors) ← NEW
 - fact_device_events (from device_logs) ← NEW

 JSONB Normalization:
 - orders.discounts → fact_order_discounts
 - orders.membership → bridge_order_membership
 - order_items.data → flattened into fact_washes
 - subscription_payments.account_info → dim_payment_methods
 - subscriptions.modes → bridge_subscription_modes

 Phase 3: Gold Layer (dbt Models - Analytics & Datamarts)

 Goal: Replicate current analytics tables, add new ones

 Analytics (gold/analytics/):
 - analytics_daily_orders
 - analytics_station_performance
 - analytics_subscription_revenue
 - analytics_subscription_usage
 - analytics_customer_metrics
 - analytics_fraud_indicators
 - analytics_daily_cashflow
 - analytics_revenue_breakdown
 - ... (all 24 current analytics tables)
 - analytics_voucher_performance ← NEW
 - analytics_device_health ← NEW

 Datamarts (gold/datamarts/):
 - datamart_active_users
 - datamart_customer_segmentation
 - datamart_revenue_dashboard
 - datamart_station_scorecard
 - datamart_fraud_report

 Phase 4: Orchestration (Dagster)

 Goal: Automate everything

 - Dagster project setup
 - Assets for ingestion (source → landing → bronze)
 - Assets for dbt (silver, gold)
 - Schedules:
   - Hourly: incremental fact loads
   - Daily 2AM: full dimension refresh
 - Sensors: trigger on new data
 - Alerting: failures, data quality issues

 Phase 5: Cutover & Validation

 Goal: Production-ready lakehouse

 - Run both systems in parallel
 - Automated comparison (old vs new output)
 - Switch API/BI to new lakehouse
 - Decommission old warehouse (keep backup)

 ---
 Project Structure

 jetx-lakehouse/
 ├── docker-compose.yml          # MinIO, Dagster UI
 ├── pyproject.toml              # Python dependencies
 │
 ├── ingestion/                  # Python: Source → Landing → Bronze
 │   ├── __init__.py
 │   ├── config.py               # Connection configs
 │   ├── sources/
 │   │   └── wash24h.py          # Source DB extractors
 │   ├── loaders/
 │   │   └── iceberg_loader.py   # Landing → Bronze
 │   └── utils/
 │       └── incremental.py      # Watermark tracking
 │
 ├── warehouse/                  # dbt project
 │   ├── dbt_project.yml
 │   ├── profiles.yml
 │   ├── models/
 │   │   ├── staging/            # Light transforms on bronze
 │   │   │   └── wash24h/
 │   │   ├── silver/
 │   │   │   ├── core/           # Shared dimensions
 │   │   │   └── wash/           # Wash facts
 │   │   └── gold/
 │   │       ├── analytics/
 │   │       └── datamarts/
 │   ├── macros/
 │   ├── tests/
 │   └── seeds/                  # Static reference data
 │
 ├── orchestration/              # Dagster
 │   ├── __init__.py
 │   ├── assets/
 │   │   ├── ingestion.py
 │   │   └── dbt_assets.py
 │   ├── schedules.py
 │   ├── sensors.py
 │   └── resources.py
 │
 ├── tests/                      # Integration tests
 │
 └── docs/                       # Documentation
     └── PROJECT_CONTEXT.md      # (existing)

 ---
 Infrastructure

 Development (Local/Droplet)

 MinIO:      localhost:9000 (S3-compatible)
 DuckDB:     Embedded (in-process)
 Dagster:    localhost:3000 (UI)
 Metabase:   localhost:3001 (BI)

 Production

 S3:         s3://jetx-lakehouse/
 DuckDB:     Same server (96GB RAM)
 Dagster:    Same server or separate

 Server Requirements

 - Current: 96GB RAM (more than enough)
 - Storage: ~50GB SSD for MinIO data (grows over time)
 - CPU: Multi-core for DuckDB parallelism

 ---
 Cost Estimate

 | Component          | Dev              | Production       |
 |--------------------|------------------|------------------|
 | Storage (MinIO/S3) | $0 (local)       | ~$5-10/month     |
 | Compute            | Existing Droplet | Existing Droplet |
 | DuckDB             | $0 (embedded)    | $0 (embedded)    |
 | Dagster            | $0 (self-hosted) | $0 (self-hosted) |
 | Metabase           | $0 (self-hosted) | $0 (self-hosted) |
 | Total              | $0               | ~$10-20/month    |

 ---
 Success Criteria

 1. Phase 0 Complete: Query dim_stations from Iceberg via DuckDB
 2. Phase 1 Complete: All source tables in bronze layer
 3. Phase 2 Complete: All dims/facts in silver, matching current warehouse
 4. Phase 3 Complete: All analytics/datamarts in gold, matching current
 5. Phase 4 Complete: Fully automated via Dagster
 6. Phase 5 Complete: Production traffic on lakehouse, old system decommissioned

 ---
 Risks & Mitigations

 | Risk                          | Mitigation                      |
 |-------------------------------|---------------------------------|
 | JSONB extraction complexity   | Start simple, iterate           |
 | dbt-duckdb + Iceberg maturity | Test early, have Trino fallback |
 | Data quality differences      | Automated comparison tests      |
 | Performance at scale          | Partitioning from day 1         |

 ---
 Open Questions (Resolved)

 - Iceberg vs Delta vs Hudi? → Iceberg (best open ecosystem)
 - Trino vs DuckDB? → DuckDB (96GB RAM, years of runway)
 - Snowflake? → No (cost, not needed at this scale)
 - Ingest from source or warehouse? → Source (richer data)
 - Landing zone from day 1? → Yes