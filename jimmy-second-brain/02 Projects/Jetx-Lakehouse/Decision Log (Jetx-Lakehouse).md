---
type: note
date: 2026-04-20
project: Jetx-Lakehouse
author: claude
---

# Decision Log — Jetx-Lakehouse

*Running log of decisions from 2026-04-20 forward. For decisions before this date, see `references/DECISION_LOG.md` (archived, frozen).*

Each entry: date, decision, why, alternatives considered, status (in effect / superseded).

---

## 2026-04-20 — Strip Metabase from the stack

**Decision:** Remove Metabase from `docker-compose.yml`; delete `docker/metabase/`.
**Why:** Metabase had locking conflicts with DuckDB and the team was already running SQL-first via CLI. Maintaining it was cost without payoff.
**Alternatives considered:** Keep it for ad-hoc dashboards; swap for Superset or Hex.
**Status:** In effect. Open question — need to decide if PowerBI + CLI is enough long-term (see open problem #8).

---

## 2026-04-20 — New ingestion pattern: PG → Parquet → Iceberg

**Decision:** Insert a Parquet staging step (`raw/`) between PostgreSQL and the Iceberg bronze layer. DuckDB + `postgres_scanner` + `httpfs` does the extraction; `load_parquet_to_iceberg()` appends to bronze with `_ingested_at` + `_source_db` stamps.
**Why:** Cleaner separation of extract vs. load, easier to replay, and `COPY ... TO s3 (FORMAT PARQUET)` is much faster and simpler than row-by-row inserts. Stamps give provenance without needing separate audit tables.
**Alternatives considered:** Direct PG → Iceberg, Airbyte/Meltano, Fivetran.
**Status:** In effect. 20 `raw_*` + 20 `bronze_*` Dagster assets in production.

---

## 2026-04-20 — Compose reduced to dagster-webserver + dagster-daemon

**Decision:** Strip the compose file to just the two Dagster services; MinIO is reached via `host.docker.internal`.
**Why:** Lower operational surface on the dev server; MinIO runs natively on the host and doesn't need to be containerized.
**Alternatives considered:** Keep MinIO in compose, run Dagster bare-metal.
**Status:** In effect.

---

## 2026-04-20 — Split env files: laptop vs. dev server

**Decision:** `.env.example` targets the dev server over Tailscale IP (for laptop use); `.env.dev.example` uses `host.docker.internal` (for the dev server itself). `.env.dev` and `.env.prod` are git-ignored.
**Why:** Same codebase, different endpoints depending on where it runs. Tailscale IP is stable; `host.docker.internal` is the right local shortcut on the dev server.
**Alternatives considered:** Single env with a `DEV_MODE` flag.
**Status:** In effect.

---

## 2026-04-20 — CLAUDE.md as project charter / architect role

**Decision:** Repo-level `CLAUDE.md` defines Claude's role as architect; `.claude_instructions` deleted.
**Why:** Consolidate governance into one file; align with the `.agent_sync/` multi-agent pattern (Builder / Orchestrator / Notes).
**Alternatives considered:** Keep the old `.claude_instructions`, split across multiple files.
**Status:** In effect.

---

*Add new entries above this line.*

## See Also
- [[Project Overview (Jetx-Lakehouse)]]
- [[Current Architecture (Jetx-Lakehouse)]]
- [[Roadmap (Jetx-Lakehouse)]]
