---
type: problems
date: 2026-04-20
project: Jetx-Lakehouse
author: claude
---

## Goal
Wash24h/JetX data warehouse — a medallion lakehouse (Bronze → Silver → Gold) that consolidates 20+ PostgreSQL source tables across operations, subscriptions, vouchers, customers, vehicles, and reference data into a single analytics-grade store, feeding FinOps, BI (PowerBI + CLI), and downstream apps.

## Why
The business is 17+ stations, ~42K customers, ~5K washes/month, and growing. Ad-hoc Pandas ETL and raw PG queries are unauditable and fragile; the medallion pattern with dbt gives tested, documented, reproducible transforms, Iceberg gives immutable archival, and Gold pre-aggregates power dashboards, APIs, and downstream projects like FinOps. Series A readiness demands numbers that hold up to audit.

## Tangible Outcomes

**Layers**
- Bronze: 20 Iceberg tables in MinIO S3, fed by the new PG → Parquet (`raw/`) → Iceberg (`bronze/`) pipeline via DuckDB + `postgres_scanner` + `httpfs`; rows stamped with `_ingested_at` and `_source_db`
- Silver: 8 dimensions + 5 facts in DuckDB, built by dbt; full-refresh on run
- Gold: 13 business-ready tables in DuckDB (revenue, station perf, MRR, etc.); 11 more specced

**Orchestration**
- 40 Dagster assets total — 20 `raw_*` + 20 `bronze_*` via a factory pattern
- Jobs chain `raw_* → bronze_*` through `AssetIn` deps
- 8 unit tests passing on the DuckDB extractor

**Deployment**
- Compose stripped to `dagster-webserver` + `dagster-daemon`; MinIO reached via `host.docker.internal`
- `scripts/push-dev.sh` — rsync + remote `docker compose up -d --build --remove-orphans`
- `scripts/cleanup-dev.sh` — preview-by-default cleanup, keeps MinIO, `--yes` to execute
- Env split: `.env.dev.example` (dev server) and `.env.example` (laptop via Tailscale IP)

**BI + Surfaces**
- Metabase removed from the stack
- PowerBI via ODBC against DuckDB
- CLI: `jetx-lake gold revenue|performance|mrr`

**Governance**
- `CLAUDE.md` in repo defines the architect role
- `.agent_sync/` coordinates Builder / Orchestrator / Notes across agents
- `docs/superpowers/specs/` holds active specs

## Open Problems
1. Complete the Gold layer — 11 remaining tables: `customer_segments`, `hourly_usage`, `live_snapshot`, `kpi_summary`, `vehicle_analytics`, `station_summary`, `wash_mode_performance`, `customer_behavior`, `cohort_retention`, `wallet_economy`, `voucher_performance`
2. Configure Dagster daily schedules (currently on-demand)
3. Validate PowerBI ODBC end-to-end from WSL2 DuckDB path
4. dbt tests and data contracts — none yet
5. Incremental models — all silver/gold are full-refresh today
6. Reference table editors (`ref_station_zones`, `ref_geo_vietnam`) — no CLI, SQL-only
7. Landing zone uploader CLI — schemas defined, no tool
8. Post-Metabase exploratory BI — is PowerBI + CLI enough, or do we need a replacement for ad-hoc SQL?
9. Monitoring + alerting on ingestion jobs (none)
10. Shake down the `.agent_sync/` multi-agent workflow in real sessions

## Doc Set (authoritative from 2026-04-20 forward)
- [[Project Overview (Jetx-Lakehouse)]] — this file; goal, why, outcomes, open problems
- [[Current Architecture (Jetx-Lakehouse)]] — the real stack right now
- [[Roadmap (Jetx-Lakehouse)]] — sequenced plan through open problems
- [[Decision Log (Jetx-Lakehouse)]] — decisions from this point forward
- [[Principles Found (Jetx-Lakehouse)]] — durable principles as they're discovered
- `references/` — frozen archive of pre-2026-04-20 docs; do not edit, use for history only

## Architecture Alignment

**Platform principles ([[jetx-platform-principles]]):**
- #4 Core domain in one service — the Silver `dim_stations`, `dim_customers`, `dim_vehicles` are *materialized* copies of core entities; FK semantics preserved, writes never flow back
- #5 Each app owns its own domain data — the lakehouse owns its Bronze/Silver/Gold schemas; no app reads lakehouse tables directly (they call the CLI or PowerBI against Gold)
- #9 Boring tech — DuckDB + dbt + Iceberg + MinIO over Spark/Snowflake/BigQuery

**Data-ownership channel position ([[jetx-data-ownership-principles]]):**
- **Channel 3 reader** for every source PostgreSQL database (20 tables across 6 domains) — the lakehouse subscribes to the source-of-truth, never writes back
- **Channel 3 publisher** of analytics-grade Gold tables — FinOps, PowerBI, and CLI are downstream consumers
- **Never Channel 1 or 2** — no sync APIs, no command topics; purely read + transform + publish

## See Also
- [[Project Overview (FinOps)]] — primary downstream consumer; Gold batch 3 feeds the P&L engine
- [[jetx-platform-principles]] · [[jetx-data-ownership-principles]] · [[00-architecture]]
- [[Principles Found (Jetx-Lakehouse)]] — durable principles discovered while building this project
