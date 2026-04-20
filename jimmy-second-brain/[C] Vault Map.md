---
type: note
date: 2026-04-20
author: claude
---

# Vault Map

*A Map-of-Content for the whole vault. One hop to anything important. Rebuild this when the shape of the vault changes.*

---

## The Spine (read these to understand how JetX/Wash24h is architected)

- [[jetx-platform-principles]] — 10 rules every app obeys (identity, data ownership, integration, etc.)
- [[jetx-data-ownership-principles]] — three-channel model for how data flows between apps, queues, and the DW
- [[00-architecture]] — enterprise framework overview (why .NET 9 + SvelteKit + Keycloak + PG17 on self-hosted infra)
- [[README]] (architecture-wash24h-ent-system) — reading order + Quick Reference table for the framework docs

---

## Active Projects

### [[Project Overview (Identity)|Identity]]
Keycloak as the single identity provider for every JetX app — realm, capability RBAC, MFA, SSO. Spine service.
- [[Project Overview (Identity)]] · [[Principles Found (Identity)]]
- Implementation spec: [[01-keycloak-setup]]

### [[Project Overview (FinOps)|FinOps]]
Consolidate financial data into a single source of truth — reconciliation, VAT compliance, P&L, Xero push. **First app to implement the full architecture end-to-end.**
- [[Project Overview (FinOps)]] · [[Principles Found (FinOps)]]
- Source docs: [[brief]] · [[finops-brd]] · [[finops-deep-spec]]

### [[Project Overview (Jetx-Lakehouse)|Jetx-Lakehouse]]
Medallion data warehouse (Bronze → Silver → Gold) feeding FinOps and BI.
- [[Project Overview (Jetx-Lakehouse)]] · [[Current Architecture (Jetx-Lakehouse)]] · [[Roadmap (Jetx-Lakehouse)]] · [[Decision Log (Jetx-Lakehouse)]] · [[Principles Found (Jetx-Lakehouse)]]

---

## Enterprise Framework (in `06 Thoughts and Ideas/architecture-wash24h-ent-system/`)

Read in order:

1. [[00-architecture]] — stack rationale, component map, repo layout
2. [[01-keycloak-setup]] — identity provider (realm, groups, roles, clients)
3. [[02-dotnet-backend]] — Clean Architecture, JWT validation, CQRS
4. [[03-sveltekit-frontend]] — BFF pattern, OIDC flow, typed API client
5. [[04-deployment]] — Incus, systemd, Caddy, monitoring, backups
6. [[05-conventions]] — code style, git, migrations, secrets

Principles:

- [[jetx-platform-principles]]
- [[jetx-data-ownership-principles]]

---

## How Everything Connects

```
              jetx-platform-principles
              jetx-data-ownership-principles
                          │  (rules every app & data flow obeys)
                          ▼
     ┌────────────────────┴────────────────────┐
     │                                         │
     ▼                                         ▼
  Enterprise Framework                    Active Projects
  (00–05 + README)                        ├─ Identity (Keycloak)
  .NET / SvelteKit / Keycloak             │     ▲
  Caddy / Incus / PG17                    │     │  JWT + capability scopes
                                          │     │
                                          ├─ FinOps (first end-to-end consumer)
                                          │     │
                                          │     │  reads Gold tables
                                          │     ▼
                                          └─ Jetx-Lakehouse
                                                (Bronze / Silver / Gold)
                                                reads from all source
                                                systems per Channel 3
```

---

## How to Work in This Vault

- **Start of day:** say "good morning" — reads recent logs and recommends the most important thing
- **Start a new project:** say "new project" — scaffolds folder, Project Overview, Principles Found, pins + icons, and updates [[CLAUDE]]
- **End of day:** say "/eod" — logs the session to `01 Daily Logs/C Logs/` and keeps [[CLAUDE]]'s Folder Structure accurate
- **First time only:** "set up my second brain" (already done)

See [[Skills]] for the full trigger list, and [[CLAUDE]] for the project-level context Claude reads at the start of every session.

---

## Housekeeping

- [[CLAUDE]] — project context file (for Claude)
- [[README]] (root) — system overview for humans
- [[Skills]] — installed skills and trigger words
- `04 Templates/` — [[template-daily]] · [[template-idea]] · [[template-note]] · [[template-solution]]

---

## Architecture Alignment Rule

Every active project must have an **Architecture Alignment** section in its `Project Overview` that:
1. Names the load-bearing principles from [[jetx-platform-principles]]
2. Declares its data-ownership channel position per [[jetx-data-ownership-principles]]
3. Lists upstream / downstream project dependencies

Projects don't invent their own rules. If a project needs to break a principle, that deviation gets logged in its `Decision Log` with the reasoning — never silently. See [[CLAUDE]] for the full rule.

---

## When to Rebuild This Map

- New project added
- A project finishes or gets archived
- A new "spine" doc appears in Thoughts and Ideas (something that many projects would reference)
- Roughly every month as a consolidation pass
