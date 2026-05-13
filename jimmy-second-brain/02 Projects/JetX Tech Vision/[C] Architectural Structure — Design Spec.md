---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-13
sub-project: Architectural Structure
---

# Architectural Structure — Design Spec (v1.0)

> **Proposal.** The shape of the JetX system: which apps exist, what each one owns, how they're deployed, how they communicate, how the topology evolves through Y1 → Y3. Builds on [[jetx-platform-principles]] (rules), [[jetx-data-ownership-principles]] (channel patterns), and [[[C] Platform & Stack Reference — Design Spec]] (component inventory). Once Jimmy approves, the shape promotes into [[Project Overview (JetX Tech Vision)]] and informs every new app's architecture-alignment block.

## 1. What this is

The named map of JetX's apps, the spine services that connect them, and the deployment topology that hosts them. This doc names *the pieces*; the principles docs name *the rules*; the stack reference names *the components*. New apps slot into this shape; they don't reinvent it.

## 2. The macro shape — one portal, many apps, shared spine

```
                  ┌──────────────────────────────────────────┐
                  │              ONE PORTAL                  │
                  │   (SvelteKit shell — nav, identity,      │
                  │    notifications, app launcher)          │
                  └──────────────────────────────────────────┘
                                     │
       ┌─────────────────────────────┴─────────────────────────────┐
       │                                                            │
       ▼                                                            ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  APP A      │  │  APP B      │  │  APP C      │  │   APP n     │
│  FinOps     │  │  Tickets    │  │  CRM        │  │   ...       │
│  (PG + .NET │  │  (PG + .NET │  │  (PG + .NET │  │   (same)    │
│   + Svelte) │  │   + Svelte) │  │   + Svelte) │  │             │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
       │                │                │                │
       └────────────────┴────────┬───────┴────────────────┘
                                 │
                                 ▼
                   ┌──────────────────────────────┐
                   │       SHARED SPINE           │
                   │  - Keycloak (identity)       │
                   │  - platform-core (entities)  │
                   │  - NATS JetStream (events)   │
                   │  - MinIO (files)             │
                   │  - Postgres 17 (per-app DBs) │
                   │  - Caddy (TLS + routing)     │
                   │  - Tailscale (network)       │
                   │  - Prometheus + Grafana      │
                   └──────────────────────────────┘
```

**Three layers, three responsibilities:**

- **Portal.** Single SvelteKit shell. Owns navigation, identity glue, notifications, app launcher. Hosts apps via iframe → path-mount → module-federation evolution (see §8).
- **Apps.** Independent units. Each owns its own repo, database, release cycle. Talks to the spine, not to other apps directly (for writes), except via change events.
- **Spine.** Boring infrastructure. Nothing app-specific lives here. Identity, master entities, broker, file storage, network, observability.

This is the literal expression of [[jetx-platform-principles]] #1: *apps stay independent; the platform is the spine.*

## 3. App inventory — what exists, what's named, what's coming

Each row is a deployable application. Each app owns its domain data (per [[jetx-platform-principles]] #5), runs on the same stack ([[[C] Platform & Stack Reference — Design Spec]] §2), and integrates via the three channels ([[jetx-data-ownership-principles]]).

| App | Owns (system of record for) | Channel position | Status | Project doc |
|---|---|---|---|---|
| **platform-core** | User, Site, Customer, Vehicle, Organization | Source for core entities; emits Channel 3 | Spine; in-flight | [[jetx-platform-principles]] #4 |
| **Keycloak** | Auth, MFA, groups, capabilities | Source for identity; emits Channel 3 (login events) | Spine; live | [[Project Overview (Identity)]] |
| **Portal** | Nothing (composes others) | N/A | Y1 build | (this doc) |
| **FinOps** | Invoices, reconciliations, P&L, VAT | Source; Channel 3 sink (settlements, EasyInvoice, ops events); Xero downstream | In-flight | [[Project Overview (FinOps)]] |
| **Lakehouse** | Gold tables, dim tables | Channel 3 reader for everything; never writes back to sources | In-flight | [[Project Overview (Jetx-Lakehouse)]] |
| **Tickets** | Internal tickets / runbooks / issues | Source for tickets | Launching Y1 | [[Project Overview (Jetx-Tickets)]] |
| **CRM** | Driver identity + vehicle profile + ops state | Source for drivers/vehicles (split with platform-core — see §4); emits Channel 3 | Y1 build (v1 EOQ3 '26) | (new project doc TBD) |
| **Loyalty** | Earn-burn rules, balances | Source for balances; emits Channel 3 (balance changes) | Y1 build | (new) |
| **JetX Coins** | Currency ledger | Source for ledger; emits Channel 3 (mint/burn events) | Y1 build | (new) |
| **Marketing tools** | Campaigns, segments, send logs | Reader of CRM; emits Channel 3 (campaign executions) | Y1 build | (new) |
| **Multi-services** | Service catalog, pricing, checkout | Source for catalog; integration point for Vending + future services | Y1 build | (new) |
| **Vending Machines** | Machine inventory, SKUs, restock state, vend events | Source for machine state; emits Channel 3 (vend events) | Y1 build | (new) |
| **IoT Platform** | Device registry, telemetry, command/control | Source for device state; ingests telemetry into Channel 3 | Y1 build | (new) |
| **Vehicles Detection** | LPR matches, sighting events | Channel 2 writer (driver-last-seen commands to CRM); emits Channel 3 (sighting events) | Y1 build | (new) |
| **Camera (future)** | Streams, video events | Source; emits Channel 3 (events) | Y2+ | (deferred) |
| **Fleet (future)** | Fleet customers, fleet vehicles | Source (likely overlaps CRM — needs design) | Y3+ | (deferred) |

**Naming convention:** each new app gets a `Project Overview (<App>).md` in `02 Projects/`, follows the JetX architecture-alignment block from CLAUDE.md.

## 4. Entity ownership map — who owns what

The cardinal data ownership rule: one app owns each entity. No duplicates. (Per [[jetx-data-ownership-principles]] #1 + [[jetx-platform-principles]] #5.)

| Entity | Owner | Why this owner |
|---|---|---|
| **User (employee/operator)** | platform-core | Cross-app entity (#4) |
| **Auth credentials / session** | Keycloak | Identity in one place ([[jetx-platform-principles]] #2) |
| **Site (JetX-operated location)** | platform-core | Cross-app entity |
| **Organization (JetX-internal org units)** | platform-core | Cross-app entity |
| **Driver (Customer C)** | CRM | Driver relationship is CRM's job; platform-core holds the FK reference but CRM is the source-of-truth |
| **Vehicle** | CRM | Same as Driver — CRM is system of record |
| **Loyalty balance** | Loyalty app | Domain-specific; Loyalty owns its rules + ledger |
| **JetX Coins balance** | JetX Coins app | Separate domain from loyalty points; Coins app owns the currency ledger |
| **Service catalog item** | Multi-services | Catalog is its own concern |
| **Wash transaction** | (existing wash codebase, refactored onto Modular Platform Y1) | Source-of-truth stays in the wash domain |
| **Vending transaction** | Vending Machines | Source-of-truth for vend events |
| **Service booking** | Multi-services | Centralized booking flow |
| **Campaign / segment** | Marketing tools | Marketing-owned |
| **Device (machine, charger, camera)** | IoT Platform | Device registry centralized |
| **LPR match event** | Vehicles Detection | Match-event source |
| **Invoice** | FinOps | Per [[jetx-platform-principles]] #5 example |
| **Ticket** | Tickets | Per #5 |
| **Partner operator (future, Y2+)** | TBD — Partner Operations app | Will be a new app in Y2 |

**Driver / Customer / Vehicle clarification:** platform-core holds the FK + a thin canonical row for cross-app reference. CRM holds the full driver and vehicle profile (preferences, visit history, contact channels, behavior). Apps that need driver info call CRM; apps that just need a stable ID use platform-core's FK.

This boundary is the most likely to drift. CRM and platform-core both touch the customer/vehicle world. The discipline: platform-core stays minimal (ID + canonical fields used by 3+ apps); CRM grows wide (everything else).

## 5. Communication patterns — the three channels in practice

Per [[jetx-data-ownership-principles]]. Restated here as architecture, not just principles.

### 5.1 Channel 1 — Sync API (human-waiting)

- Caller: any app (or external client) with a human waiting.
- Endpoint: target app's REST API at `api.wash24h.com/<app>/...`.
- Auth: Bearer JWT validated against Keycloak JWKS.
- Latency target: <200ms.
- Examples: portal UI editing a customer; admin assigning a capability; finance correcting an invoice.

### 5.2 Channel 2 — Command topic (system-to-system writes)

- Caller: any app, no human waiting.
- Topic: `cmd.<target-app>.<entity>.<verb>` on NATS JetStream.
- Envelope: CloudEvents 1.0 + JSON Schema per command type.
- ACKs: durable result stream, observable but NEVER gating.
- Examples: Vehicles Detection → CRM `customer.update_last_seen`; FinOps nightly reconciliation; IoT Platform → Multi-services service-state update.

### 5.3 Channel 3 — Change events (fanout after any write)

- Emitter: every app, after every successful Channel 1 or Channel 2 write.
- Topic: `events.<source-app>.<entity>.<verb>` on NATS JetStream.
- Subscribers: Lakehouse (always); other apps that need a downstream cache.
- Envelope: CloudEvents 1.0 + JSON Schema.
- Examples: `customer.updated.v1` from CRM; `invoice.paid.v1` from FinOps; `device.telemetry.received.v1` from IoT.

### 5.4 What goes through each channel (cheat sheet)

```
A human clicks "save"               → Channel 1 (sync API)
A sensor fires + system reacts      → Channel 2 (command) + Channel 3 (event)
A nightly batch runs                → Channel 2 (commands) + Channel 3 (events)
Lakehouse needs to materialize      → Channel 3 (events)
Another app needs a cache refresh   → Channel 3 (events)
```

## 6. Deployment topology

### 6.1 Y1 — single region (Vietnam, HCMC primary)

```
┌──────────────────────────── HCMC datacenter (primary) ──────────────────────────────┐
│                                                                                      │
│   Incus host(s)                                                                      │
│   ├─ Caddy (TLS + routing)                                                           │
│   ├─ Keycloak container                                                              │
│   ├─ Postgres 17 (hcm-db-a) — keycloak DB + jetx_app DB + per-app DBs               │
│   ├─ NATS JetStream cluster                                                          │
│   ├─ MinIO                                                                           │
│   ├─ App containers (one per .NET API + frontend per app)                            │
│   ├─ Prometheus + Grafana + Alertmanager                                             │
│   └─ Solar + BESS for resilience                                                     │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘

         + Cloud / colocation overflow for cost arbitrage (hybrid cloud strategy)

         + Tailscale mesh connecting datacenter, station Pi devices, dev laptops
```

**Failure domains in Y1:**

- Single datacenter; one-host-down is the most likely failure scenario.
- Backups: pg_dump + restic to MinIO + offsite copy.
- HA goal (Y1 EOQ Q1 '27 per [[[C] 12-Month Deep Dive — Design Spec]]): PG + Keycloak + NATS in HA mode.

### 6.2 Y2 — multi-region (one new region added)

- Second region runs an instance of the spine (Keycloak read-replica or federated realm — design TBD; Postgres replica; NATS cluster).
- Apps deployed regionally; CRM + Loyalty + Coins federate across regions (one driver identity across regions).
- Cross-region change events replicate via NATS or a federation pattern.
- Latency budget: regional users hit regional spine; cross-region operations are async.

### 6.3 Y3 — multi-tenant + multi-region

- Modular Platform v3 MT-production: each tenant gets its own logical isolation (config + data scoping + capability tree), running on the shared infrastructure.
- New tenant onboarding becomes a config change (per the Y3 partner-platform goal).
- Regional + tenant axes are orthogonal — a region can host multiple tenants; a tenant can span regions.

## 7. Failure domains + isolation rules

These rules govern what fails together vs separately.

**F1. Apps are independently deployable.** A FinOps release doesn't block a Tickets release. (Per [[jetx-platform-principles]] #1 + the red-flag list.)

**F2. Spine outage degrades, doesn't crash.** Identity outage means new logins fail; existing sessions continue (JWT expiry tolerance). Broker outage means new events queue locally; sync APIs continue. PG outage = real downtime (acknowledged single point of failure until HA).

**F3. Per-app database = per-app blast radius.** A schema mistake in CRM doesn't affect FinOps.

**F4. Tenant boundaries in Y3 are isolation boundaries, not just config boundaries.** A misbehaving tenant cannot consume another tenant's resources beyond shared infra quotas.

**F5. Backup discipline.** Every app must declare RPO + RTO targets in its `Project Overview`. Spine services: RPO ≤ 1h, RTO ≤ 4h.

## 8. Integration evolution — iframe → path-mount → module-federation

Per [[jetx-platform-principles]] #6: graduate when it hurts, not before.

| Stage | What it is | When | JetX status |
|---|---|---|---|
| **Stage 1 — iframe-mounted** | Portal mounts each app in an iframe; JWTs shared via domain cookie. Zero refactoring on day one. | Default starting point | Where we are |
| **Stage 2 — path-mounted shared layout** | Portal proxies each app under a path. Same domain. Shared chrome (nav, notifications). | When iframe UX pain is real (poor deep-linking, awkward focus, mobile layout issues) | Likely Y1 EOQ4 or Y2 |
| **Stage 3 — Module Federation** | Apps publish federated modules; portal composes at the JS level. | When multiple teams fight over portal code | Likely Y3 if at all |

**Stage 2 trigger criteria** (decide in workshop / quarterly review):

- iframe deep links broken in N% of mobile flows.
- Notification UX inconsistent across apps.
- Performance degradation from iframe overhead measurable.

Don't jump to Stage 2 prematurely. Each stage is a real refactor with real cost.

## 9. Multi-tenant readiness — where the primitives land

Per [[[C] 12-Month Deep Dive — Design Spec]] §4 Modular Platform v2 (EOQ1 '27), MT primitives ship into Modular Platform. The MT-readiness shape:

```
┌──────────────────────────────────────────────────────────────┐
│                  Per-tenant config / scope                    │
│  - Tenant ID propagated through every request (JWT claim)    │
│  - Per-tenant capability subsets in Keycloak                 │
│  - Per-tenant data partitioning (schema-per-tenant OR        │
│    row-level filter — decision TBD in workshop)              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
       Every app inherits MT-readiness from Modular Platform.
       No app does its own MT logic.
```

**MT decisions deferred to the workshop (per [[[C] 12-Month Deep Dive — Design Spec]] open question #6):**

- Data isolation: schema-per-tenant vs row-level + tenant_id everywhere.
- Identity isolation: separate Keycloak realms vs shared realm with tenant claim.
- Quota / rate limit isolation: shared infra with per-tenant quotas vs separate infra per tenant.

The shape supports any of these; the choice is made when the first real partner tenant approaches in Y3.

## 10. Agent-fleet-to-app mapping

How [[[C] AI-First Framework — Design Spec]]'s fleet operators map to this app inventory:

| Fleet (operator role) | Apps owned | Notes |
|---|---|---|
| **FinOps lead** | FinOps | Single-app fleet; mature in-flight track |
| **Identity lead** | Keycloak + platform-core (auth slice) | Spine ownership; high-leverage |
| **Lakehouse lead** | Lakehouse | Single-app fleet; mature in-flight track |
| **Tickets lead** | Tickets | Single-app fleet |
| **Platform lead** | Modular Platform + Multi-services + Portal | Coupled trio; spine-adjacent |
| **Hardware lead** | Vending + IoT + Vehicles Detection | Hardware-touching cluster |
| **Relationship lead** | CRM + Loyalty + Coins | Driver-relationship coupled trio |
| **Growth lead** | Marketing tools | Single-app; depends on Relationship lead |
| **Platform / SRE** | All spine services (Caddy, NATS, MinIO, observability) | Cross-cutting |

The agent fleet doesn't change the architecture; it changes how the architecture gets built. App boundaries are owned by fleet operators; cross-app contracts are owned jointly (and reviewed by both fleets).

## 11. Open questions

1. **Driver/Customer/Vehicle split between platform-core and CRM** — concrete entity-by-entity boundary. Needs a one-page sub-spec before CRM v1 ships.
2. **MT data isolation pattern** (schema-per-tenant vs row-level) — workshop question #6 from 12-Month Deep Dive.
3. **MT identity pattern** (separate realms vs shared realm with claim) — Y2 decision.
4. **Portal Stage-2 trigger** — measurable iframe pain criteria.
5. **Y2 regional federation pattern** — Keycloak realm replication strategy; NATS cross-region topology.
6. **IoT broker placement** — share NATS JetStream with the rest, or run a separate MQTT broker for device traffic? (Cross-referenced with [[[C] Platform & Stack Reference — Design Spec]] open question #5.)
7. **Camera / Fleet apps** — when these surface, who owns the entity boundary against CRM (drivers) and platform-core (sites)?

## 12. Versioning

- **v1.0 — 2026-05-13.** Initial; codifies current shape + projected evolution to Y3.
- **Revisit triggers:**
  - A new app is added → §3 updated.
  - An entity ownership boundary moves → §4 updated.
  - A failure-domain rule is broken in production → §7 audit.
  - Y1 EOQ → recalibrate stage of integration evolution (§8).
  - Y2 region opens → §6.2 promoted from projection to current state.

### Change log

- 2026-05-13 — v1.0 initial.

## 13. Promotion checklist (Jimmy)

When you approve this spec:

- [ ] Add a `## Architectural Structure (v1.0)` section to [[Project Overview (JetX Tech Vision)]] referencing §2 macro shape + §3 app inventory; full doc stays here.
- [ ] Create `Project Overview (<App>).md` stubs for the Y1 new apps (CRM, Loyalty, JetX Coins, Marketing tools, Multi-services, Vending Machines, IoT Platform, Vehicles Detection, Portal). Each stub gets the architecture-alignment block + a placeholder for fleet-operator assignment.
- [ ] Decide the platform-core/CRM entity split (open question #1) before CRM v1 design ships.
- [ ] Set this note's `status` to `actioned`.
- [ ] *(Optional)* Add a [[Decision Log (JetX Tech Vision)]] entry: `2026-05-13 — Architectural structure v1.0: 14 apps + spine; Y1 stays single-region single-tenant; multi-tenant primitives Y1 EOQ1 '27; first new region Y2.`

## 14. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project.
- [[[C] Platform & Stack Reference — Design Spec]] — the component inventory this structure runs on.
- [[[C] Governing Principles — Design Spec]] — the rules layer above this shape.
- [[[C] 12-Month Deep Dive — Design Spec]] — Y1 build plan that ships 9 of these apps.
- [[[C] 3-Year Roadmap — Design Spec]] — when each future app lands.
- [[[C] AI-First Framework — Design Spec]] — agent-fleet-to-app mapping in §10.
- [[jetx-platform-principles]] — Pillar B (spine + app boundaries).
- [[jetx-data-ownership-principles]] — Pillar C (channels).
- [[00-architecture]] — current canonical stack snapshot.
- [[Project Overview (FinOps)]], [[Project Overview (Identity)]], [[Project Overview (Jetx-Lakehouse)]], [[Project Overview (Jetx-Tickets)]] — in-flight projects mapped in §3.
