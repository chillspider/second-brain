---
type: note
date: 2026-04-20
project: Jetx-Lakehouse
author: claude
---

# Roadmap — Jetx-Lakehouse

*Sequenced plan against the open problems in `Project Overview (Jetx-Lakehouse).md`. Update as priorities shift or items land.*

## Now (this sprint)

- [ ] **Dagster daily schedules** — set up nightly `raw_* → bronze_*` runs + dbt silver/gold; pick a time, wire alerting on failure (open problem #2)
- [ ] **PowerBI ODBC end-to-end check** — prove the DuckDB ODBC path works from WSL2, lock down which users need it, document (open problem #3)
- [ ] **dbt tests — baseline** — `not_null` + `unique` on every dim PK, `relationships` on every fact FK; get CI green before adding more models (open problem #4)

## Next (1–2 sprints out)

- [ ] **Gold layer batch 1 — customer intelligence** — `customer_segments`, `customer_behavior`, `cohort_retention` (open problem #1)
- [ ] **Gold layer batch 2 — station ops** — `station_summary`, `hourly_usage`, `wash_mode_performance`, `live_snapshot` (open problem #1)
- [ ] **Incremental silver models** — convert `fct_orders` and `fct_transactions` to incremental; validate against full-refresh (open problem #5)
- [ ] **Monitoring + alerts** — Dagster sensors on run failure, row-count regression alerts on bronze loads (open problem #9)

## Later

- [ ] **Gold layer batch 3 — finance-adjacent** — `kpi_summary`, `wallet_economy`, `voucher_performance`, `vehicle_analytics` (open problem #1) — these feed FinOps directly
- [ ] **Reference-table editor CLI** — `jetx-lake ref edit station-zones` etc. to stop editing SQL/CSV by hand (open problem #6)
- [ ] **Landing zone uploader CLI** — `jetx-lake landing upload <schema> <file.csv>` with validation (open problem #7)
- [ ] **Data contracts** — formalize source → bronze schemas; fail loud on breaking changes (open problem #4)
- [ ] **Exploratory BI decision** — determine if PowerBI + CLI is enough or if we reintroduce a dashboard layer (Metabase replacement, Superset, Hex, roll our own) (open problem #8)
- [ ] **`.agent_sync/` shakedown** — run 2–3 real multi-agent sessions against this project, capture what's missing (open problem #10)

## Cross-project dependencies

- [[Project Overview (FinOps)]] is the primary downstream consumer. The Gold tables it needs (per [[finops-brd]]) are revenue reconciliation inputs, VAT-adjacent transaction data, and cost ingestion staging. Coordinate Gold batch 3 with the FinOps team's P&L engine timeline.

## Done (since 2026-04-20)

*(Move items here as they land.)*

## See Also
- [[Project Overview (Jetx-Lakehouse)]] — the open problems this roadmap sequences
- [[Current Architecture (Jetx-Lakehouse)]] — the stack each item builds against
- [[Decision Log (Jetx-Lakehouse)]] — record decisions made while executing
