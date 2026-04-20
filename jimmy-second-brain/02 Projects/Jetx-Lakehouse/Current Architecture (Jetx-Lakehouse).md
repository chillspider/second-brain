---
type: note
date: 2026-04-20
project: Jetx-Lakehouse
author: claude
---

# Current Architecture — Jetx-Lakehouse

*The authoritative snapshot of the stack as of 2026-04-20. Supersedes `references/ARCHITECTURE.md`.*

## Data Flow

```
PostgreSQL (20+ source tables across 6 domains)
          │
          │  queries/raw/{domain}/{table}.sql  (one SELECT per table)
          ▼
DuckDB extractor — postgres_scanner + httpfs
          │  COPY … TO s3 (FORMAT PARQUET)
          ▼
MinIO S3 — raw/  (Parquet staging)
          │
          │  ingestion/loaders/iceberg.load_parquet_to_iceberg()
          │    stamps _ingested_at, _source_db
          ▼
MinIO S3 — bronze/  (20 Iceberg tables — immutable, append-only)
          │
          │  dbt (dbt-duckdb) — Silver models
          ▼
DuckDB — main_silver  (8 dims + 5 facts, full-refresh on run)
          │
          │  dbt — Gold models
          ▼
DuckDB — main_gold  (13 business tables, grows to ~24 per spec)
          │
          ├─► CLI  (jetx-lake gold revenue | performance | mrr)
          └─► PowerBI (ODBC)
```

## Layers

### Bronze — MinIO S3 + Apache Iceberg
- 20 tables, one per source table, across six domains: operations, subscriptions, vouchers, customers, vehicles, reference
- Every row stamped with `_ingested_at` (UTC timestamp) and `_source_db` (which PG instance it came from)
- Append-only; archival of record

### Silver — DuckDB (`transform/lakehouse.duckdb`, schema `main_silver`)
- 8 dimensions: `dim_stations`, `dim_customers`, `dim_devices`, `dim_vehicles`, `dim_vouchers`, `dim_plans`, `dim_modes`, `dim_payment_profiles`
- 5 facts: `fct_orders`, `fct_transactions`, `fct_subscriptions`, `fct_daily_active_users`, `fct_daily_churn`
- Rebuilt on every dbt run

### Gold — DuckDB (same file, schema `main_gold`)
- Built: `gold_daily_revenue`, `gold_station_performance`, `gold_subscription_mrr` (+10 others already implemented)
- Specced but not built: `customer_segments`, `hourly_usage`, `live_snapshot`, `kpi_summary`, `vehicle_analytics`, `station_summary`, `wash_mode_performance`, `customer_behavior`, `cohort_retention`, `wallet_economy`, `voucher_performance`

## Orchestration — Dagster

- **40 assets** via factory pattern:
  - 20 × `raw_*` — one per source table, runs the `.sql` extractor
  - 20 × `bronze_*` — consume upstream `raw_*` via `AssetIn`, append to Iceberg
- Jobs: raw and bronze run together through deps (no manual chaining)
- Schedules: **not yet configured** (on-demand only for now)

## Deployment

### Compose
```
services:
  dagster-webserver
  dagster-daemon
```
That's it — Metabase stripped. MinIO is reached via `host.docker.internal` from the dev server.

### Environments
- `.env.example` — laptop; points at Tailscale IP of the dev server
- `.env.dev.example` — dev server; uses `host.docker.internal`
- `.env.dev` and `.env.prod` — git-ignored

### Scripts
- `scripts/push-dev.sh` — rsync repo to dev server, then `docker compose up -d --build --remove-orphans`
- `scripts/cleanup-dev.sh` — preview-by-default cleanup; pass `--yes` to execute; MinIO is preserved

## Surfaces

- **CLI** — `jetx-lake gold revenue | performance | mrr` (primary for ops users)
- **PowerBI** — ODBC against the DuckDB file (ad-hoc exploration)
- **Metabase** — removed

## Governance

- `CLAUDE.md` in repo defines the architect role (Claude as architect)
- `.agent_sync/` — Builder / Orchestrator / Notes files for multi-agent coordination
- `docs/superpowers/specs/` — active specs

## Testing

- `tests/ingestion/test_duckdb_extractor.py` — 8 unit tests, all passing
- No dbt tests yet
- No data contracts yet

## Known Gaps (tracked in Project Overview, open problems)

See [[Project Overview (Jetx-Lakehouse)]] for the full list.

## See Also
- [[Project Overview (Jetx-Lakehouse)]] — goal, why, open problems
- [[Roadmap (Jetx-Lakehouse)]] — sequenced plan against this architecture
- [[Decision Log (Jetx-Lakehouse)]] — why the stack looks this way
- [[jetx-data-ownership-principles]] — Channel 3 reader pattern
- [[Project Overview (FinOps)]] — primary downstream consumer of Gold
