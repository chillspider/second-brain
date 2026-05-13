---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-13
sub-project: 12-Month Deep Dive
---

# 12-Month Deep Dive — Design Spec (v1.0)

> [!summary] Proposal
> Quarter-by-quarter tactical plan for **Q3 2026 → Q2 2027**, prepared to support the internal workshop on **2026-05-16/17**. Once Jimmy approves, the matrix + capacity model promote into [[Project Overview (JetX Tech Vision)]] and a follow-up roadmap doc. Status moves to `actioned` after the workshop.

---

## 1. The year arc

**"Year of Foundation."** Build the platform right so 2027+ can layer differentiated features (charging, partner operators, multi-tenancy in production) onto a solid base. We're not chasing new service types this year — we're making the platform able to *host* any service type cleanly.

> [!info] North Star tag *(see [[[C] Vision and Mission — Design Spec]])*
> - Advances **"platform"** — modular architecture is the load-bearing word made real.
> - Advances **"trusted"** — CRM + Loyalty + JetX Coins make the driver relationship a system of record, not a side effect.
> - Advances **"AI-powered"** — the team itself is an AI-augmented coding team; hybrid cloud hardening prepares spine for AI workloads.

---

## 2. The three pillars (+ in-flight spine)

| Pillar                                  | What's in it                                                                                                                                                                  | Why now                                                                                                       |
| :-------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------ |
| **P1 — Modular Platform Architecture**  | Refactor wash codebase onto an expandable platform: multi-service-ready, multi-tenant-ready, connected. Houses Multi-services, Vending Machines, IoT Platform, Vehicles Detection. | Without this, every new service in 2027+ is a rewrite. Pillar that pays compound interest.                    |
| **P2 — Customer Relationship Layer**    | CRM + Loyalty + JetX Coins + Marketing tools. The driver relationship gets a real system of record.                                                                          | Trust is the moat; trust needs a relationship; relationship needs a system of record.                         |
| **P3 — Hybrid Cloud Hardening**         | Cost optimization + robustness + observability. Spine gets production-grade and cheaper.                                                                                     | We are about to host many services + driver data. Infra has to be cheap, robust, and observable.              |
| **Spine (in-flight)**                   | [[Project Overview (FinOps)]], [[Project Overview (Identity)]], [[Project Overview (Jetx-Lakehouse)]], [[Project Overview (Jetx-Tickets)]]                                   | Already mid-flight. Each one hits a major milestone in-window and graduates from "project" to "platform service." |

---

## 3. The cadence — every 6 months ships a major release

Build philosophy: **parallel-build, not sequential.** Every critical module gets v1 by end of Q3 2026. Then v2 by end of Q1 2027. Then v3 by end of Q3 2027 (outside this window).

| Release point                              | Meaning                                                                                                            |
| :----------------------------------------- | :----------------------------------------------------------------------------------------------------------------- |
| **EOQ3 2026 — v1 line**                    | Every critical module is real and in production for at least 1 real use case. The "everything-is-real" anchor.    |
| **EOQ1 2027 — v2 line**                    | Every module covers its full intended scope. Polish + scale + edge cases done.                                    |
| **EOQ3 2027 — v3 line** *(out of window)*  | Differentiation layer added — partner-readiness, charging, advanced AI features.                                 |

### The v1-discipline rule (the contract that makes Q3 possible)

> [!important] v1 definition
> **v1 = the smallest thing that is real and in production for at least one real use case.** Not "good enough for everyone." Not "feature-complete." Not "polished UI." Real, in production, one use case proven end-to-end. Polish lives in v2.
>
> Without this rule, the Q3 v1 line collapses. With this rule, the Q3 line is the workshop's contract.

### Build mindset — build, test, fix, test, build, improve

The team operates a continuous loop, not a waterfall. Each module is always in some part of the loop:

> [!tip] The loop
> **Build → test → fix → test → build more → improve. Repeat in production.**

This is why "v1" doesn't mean "finished." v1 is just the moment the loop starts running with real users on the live system. The loop never stops; v2 and v3 are checkpoints, not endpoints. This mindset is what makes the AI-augmented team fast: agents build the next iteration while humans steer the next test cycle. Waterfall thinking ("design fully, then build fully, then test fully, then ship") destroys this leverage.

**What this means in practice:**

- **No long-lived feature branches.** Ship behind feature flags; iterate in main.
- **Production is the integration environment.** Staging exists for safety gates, not for prolonged QA cycles.
- **Bugs are routine, not exceptional.** Each module budgets for live-prod iteration loops *before* it's "done."
- **Polish is earned, not planned.** A module gets polish in the iteration it shows up needing polish — not on a schedule.

This mindset also justifies §5 constraint #6: live-production iteration is the dominant bottleneck because **the loop only runs at calendar speed in real prod**, no matter how fast the AI fleet codes.

---

## 4. The matrix

| Module                              | **Q3 2026** (Jul–Sep) — v1                                                                          | **Q4 2026** (Oct–Dec)                                          | **Q1 2027** (Jan–Mar) — v2                                                                   | **Q2 2027** (Apr–Jun)                                          |
| :---------------------------------- | :-------------------------------------------------------------------------------------------------- | :------------------------------------------------------------- | :------------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| **Modular Platform** *(P1)*         | **v1**: service catalog primitive + connected-platform contracts; wash refactored onto it           | Stabilize, prod-test, regression coverage                      | **v2**: multi-tenant primitives (config isolation, tenant scoping — not yet productized)     | Stabilize, doc'd for next module to onboard                    |
| **Multi-services** *(P1)*           | **v1**: wash + detail + vending in one catalog with pricing/checkout                                | Edge cases, more SKU types, refund flows                       | **v2**: oil/tire/battery as service stubs (catalog-ready, not yet operationally live)        | Stabilize                                                      |
| **Vending Machines** *(P1)*         | **v1**: 3 sites live, drinks + car-care SKUs, payment integrated into wash app                      | More sites, inventory dashboards, restock workflows            | **v2**: dynamic pricing, recommendation engine                                               | Stabilize                                                      |
| **IoT Platform** *(P1)*             | **v1**: 1 device type (wash machine OR vending) sending telemetry + basic cmd/control               | Add 2nd device type                                            | **v2**: edge processing, OTA firmware updates                                                | Stabilize                                                      |
| **Vehicles Detection** *(P1)*       | **v1**: LPR at 2 pilot sites → driver lookup in CRM                                                 | Scale to 10 sites                                              | **v2**: behavioral profile (visit cadence, dwell time, service preference)                   | Stabilize                                                      |
| **CRM** *(P2)*                      | **v1**: driver identity + vehicle profile + ops console (read/write)                                | Full station coverage; backfill historical drivers             | **v2**: segments + journeys + lifecycle automation                                           | Stabilize                                                      |
| **Loyalty** *(P2)*                  | **v1**: earn-burn rules + balance ledger (wash service only)                                        | Campaign engine, more redemption surfaces                      | **v2**: tiers + gamification + referral mechanics                                            | Stabilize                                                      |
| **JetX Coins** *(P2)*               | **v1**: mint + burn at wash, single-currency ledger                                                 | Burn at vending + detail                                       | **v2**: full cross-service spend rail                                                        | Stabilize                                                      |
| **Marketing tools** *(P2)*          | **v1**: push + SMS + in-app + basic segments (powered by CRM)                                       | A/B testing, campaign analytics                                | **v2**: automation + personalization                                                         | Stabilize                                                      |
| **Hybrid Cloud Hardening** *(P3)*   | Cost audit r1 + first cuts implemented (reserved capacity, idle shutdown, image right-sizing)       | HA for spine services (PG, Keycloak, message bus)              | Cost audit r2 + unified observability (metrics/logs/traces)                                  | SLOs locked per service; runbooks; on-call rotation live       |
| **FinOps** *(in-flight)*            | Daily-use complete for finance team                                                                 | All-station coverage; auto-reconciliation                      | Audit-grade prep; Xero parity tests                                                          | Series-A ready                                                 |
| **Lakehouse** *(in-flight)*         | Silver complete                                                                                     | Gold + PowerBI integration                                     | dbt tests + audit-grade lineage                                                              | Stable; Iceberg tuned for cost                                 |
| **Identity** *(in-flight)*          | GA across all current apps                                                                          | MFA tightened; session policy locked                           | Capability schema v2 (multi-tenant-ready)                                                    | MT-readiness done; hand-off doc to platform team               |
| **Tickets** *(in-flight)*           | Launched, in active internal use                                                                    | Lessons captured → patterns library                            | Patterns applied to next-app scaffolding                                                     | Stable; reference implementation                               |

---

## 5. Capacity — AI-augmented team

This team is **not human-throughput-bounded.** Engineers run fleets of AI coding agents; each engineer's effective output is several human-engineers' worth. The 6-10 named engineers represent **fleet operators**, not solo coders.

> [!warning] Real constraints — not raw code volume
> 1. **Spec clarity** — agents only build what specs describe. Bad specs → wasted runs. The team's leverage is in writing sharp specs and design docs, then steering agent fleets through them. This deep-dive is itself an example.
> 2. **Code review velocity** — humans still ship the merge button. Review is the bottleneck, not authorship. Workshop should address: review patterns, paired-review with agents, automated correctness gates.
> 3. **Integration testing** — parallel-built modules collide at integration boundaries. The Modular Platform's contracts (P1 v1) are what make parallel module work integratable.
> 4. **Deployment / ops capacity** — shipping 9 net-new v1s to production in Q3 means 9 first-time prod incidents to absorb. Hybrid Cloud Hardening (P3) and on-call (Q2 2027) buy us margin.
> 5. **Architectural coherence** — agents will happily produce code that drifts from the spine. Humans hold architectural alignment. [[jetx-platform-principles]] and [[jetx-data-ownership-principles]] are the alignment surface.
> 6. **Live-production iteration loop** — v1 = "real and in production for one use case." That means each module has to absorb bugs only real users surface. Each production iteration (deploy → observe in live traffic → diagnose → fix → re-deploy) is calendar-paced, not AI-paced. Typical loop: 1-3 calendar days per iteration. A module needs ~5-10 such iterations to reach v1-quality. **This is the dominant bottleneck.**

### Throughput model — back-of-envelope

Picking midpoint assumptions; workshop sharpens these.

| Variable                                       | Value                                            | Source                                                                                              |
| :--------------------------------------------- | :----------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| Team size                                      | 8 engineer-fleets                                | midpoint of 6–10                                                                                    |
| Concurrent live-prod iterations per fleet      | 1 module at a time                               | constraint #6 — one fleet can't iterate two live modules simultaneously without confusion           |
| Iteration cycle time                           | ~2 calendar days                                 | constraint #6 typical                                                                                |
| Iterations per module to reach v1              | ~7                                               | constraint #6 midpoint                                                                              |
| Window from workshop to EOQ3                   | ~20 calendar weeks (= ~100 working days)         | 2026-05-16 → 2026-09-30                                                                              |
| Net-new module v1s in window                   | 9                                                | matrix §4: Modular Platform, Multi-services, Vending, IoT, VD, CRM, Loyalty, Coins, Marketing       |
| In-flight tracks finishing milestones          | 4                                                | FinOps, Identity, Lakehouse, Tickets                                                                |

**The math:**

| Side    | Calculation                                       | Result                                              |
| :------ | :------------------------------------------------ | :-------------------------------------------------- |
| Demand  | 9 net-new × 7 iterations                          | **63 production-iteration units needed for v1**     |
| Supply  | 8 fleets × (100 days ÷ 2 days/iteration)          | **400 production-iteration slots available**        |
| Headroom | 63 ÷ 400                                          | **~6× headroom on raw throughput**                  |

> [!success] Conclusion — math is comfortably feasible
> Raw throughput is not the constraint. **Launch sequencing + integration choreography is.** The workshop's hardest output is the Q3 launch calendar — when each module enters live prod, in what order, with which integration boundaries first.

**Where it actually breaks** (not in the throughput math, but in calendar / coordination):

- **Launch staggering.** Can't take 9 net-new modules to live prod in the same week — ops absorbs them sequentially. Q3 needs a launch calendar (one module to live prod per ~1.5 weeks).
- **Integration windows.** Modules that share boundaries (CRM/Loyalty/Coins; Multi-services/Vending; Modular Platform/everything) can only deeply iterate one-at-a-time without thrashing the contracts.
- **Code review bandwidth.** 8 fleets pushing PRs at AI-augmented rate can flood reviewers. Need automated correctness gates + paired-review patterns.
- **Real-world traffic.** Some modules only iterate when real drivers/sites exercise them. Vending v1 needs ≥3 sites live before iteration data is meaningful; Vehicles Detection v1 needs ≥2 sites with cameras up.

### Suggested fleet-operator → module mapping

*(workshop redoes this with named people)*

| Engineer-fleet         | Carries                                                                                                            |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------- |
| FinOps lead            | [[Project Overview (FinOps)]] through Series-A-ready                                                               |
| Identity lead          | [[Project Overview (Identity)]] through MT-ready hand-off                                                          |
| Lakehouse lead         | [[Project Overview (Jetx-Lakehouse)]] through Iceberg-tuned                                                        |
| Tickets lead           | [[Project Overview (Jetx-Tickets)]] through reference-implementation                                               |
| Platform lead          | Modular Platform + Multi-services (both P1, deeply coupled)                                                        |
| Hardware lead          | Vending Machines + IoT Platform + Vehicles Detection (P1 hardware-touching trio)                                   |
| Relationship lead      | CRM + Loyalty + JetX Coins (P2, tightly coupled)                                                                   |
| Growth lead            | Marketing tools (P2, depends on CRM data model)                                                                    |
| Platform/SRE rotating  | Hybrid Cloud Hardening (P3, everyone owns a slice)                                                                 |

That's 9 named fleet operators on the critical surface. If the team is 6, some operators carry 2 groups (e.g., Hardware lead also runs Marketing once vending POS is stable). If the team is 10, the extras absorb review + ops + integration testing capacity (the real bottleneck).

---

## 6. The EOQ3 2026 v1 line — workshop story anchor

By end of Q3 2026 (about 4.5 months from the workshop date), JetX is operationally a different company:

> [!success] EOQ3 2026 — the "everything-is-real" moment
> - **Every driver** who interacts with JetX has a CRM record, a vehicle profile, a loyalty balance, and a JetX Coins wallet.
> - **Every JetX site** sells more than wash — at least one detail service and vending products through one catalog.
> - **Every wash machine and vending machine** is reporting telemetry through one IoT spine.
> - **Every license plate** at pilot sites resolves to a known driver automatically.
> - **The platform itself** is module-shaped, contract-defined, multi-tenant-ready architecturally.
> - **The spine** (FinOps, Identity, Lakehouse, Tickets) is in daily production use, not aspirational.

That's the one-sentence story for slide 1.

---

## 7. Non-goals (explicit drop list for 2026)

> [!warning] These are real choices, not omissions
> Each is deferred *intentionally* to keep the year focused:

| Non-goal                                                    | Why deferred                                                                                                                |
| :---------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------- |
| **Charging / EV service launch**                            | Year 2. We build IoT Platform v1 which makes chargers possible later; we don't launch chargers in 2026.                     |
| **Partner operator onboarding**                             | Year 2. We build Modular Platform v2 multi-tenant primitives which make partners possible later; we don't onboard in 2026.  |
| **Multi-tenancy in production**                             | Year 2. We build the primitives in 2026 (Modular Platform v2 in Q1 2027) but no real second tenant runs on the platform.   |
| **Geographic expansion beyond current footprint**           | Capex bet that competes with foundation work. Defer.                                                                        |
| **Series A close**                                          | Separate workstream owned by Jimmy + finance, not in this engineering plan. FinOps reaches Series-A-*ready* as a capability. |
| **Public-facing brand work** (Wash24h vs. JetX positioning) | Separate workstream, not engineering.                                                                                       |

---

## 8. Architecture alignment

This sub-project is the operational expression of [[Project Overview (JetX Tech Vision)]] for 2026. It must align to the spine; it cannot invent its own.

| Spine doc                            | How this sub-project aligns                                                                                                                                                                                                                |
| :----------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [[jetx-platform-principles]]         | Load-bearing principles: identity in one place (Identity GA), data ownership clarity (CRM as system of record for drivers, Lakehouse as analytical sink), contract-defined integration (Modular Platform v1).                              |
| [[jetx-data-ownership-principles]]   | See per-module channel position table below.                                                                                                                                                                                              |
| [[00-architecture]]                  | Every new module is .NET 9 + SvelteKit + PG17 + Keycloak per the spine. No new stack inventions in 2026; that's a v3 (2027 H2+) conversation.                                                                                              |

### Module channel positions (per [[jetx-data-ownership-principles]])

| Channel position          | Modules                                                                          |
| :------------------------ | :------------------------------------------------------------------------------- |
| **Source-of-truth**       | CRM (drivers), Multi-services (catalog), Loyalty (balances), Coins (ledger), Identity (auth) |
| **Command writers**       | Vending (orders), Marketing (campaigns), Vehicles Detection (matches)            |
| **Downstream sinks**      | Lakehouse (everything), FinOps (financial slice)                                 |

---

## 9. Open questions for the workshop

> [!question] These don't block the spec but need to land during the 2026-05-16/17 sessions

| # | Question                                                                                                                                  |
| :- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **Specific engineer-to-fleet assignments** — who carries which group of modules? Map fleet operators to named people.                     |
| 2 | **v1 scope sharpness per module** — for each module, the "smallest real thing in production" is a 1-page spec we don't yet have. Workshop afternoon = one room per pillar drafting these. |
| 3 | **Code review patterns for AI-augmented work** — what's the team's standard for agent-authored PR review? Paired-review? Automated gates? This determines the real merge throughput. |
| 4 | **Integration test strategy** — Modular Platform v1 defines the contracts. Who owns contract tests? When does the integration test suite get bootstrapped? |
| 5 | **EOQ3 launch sequence** — 9 net-new v1s landing in Q3 is a launch-coordination problem, not just an engineering one. Who runs the launch calendar? |
| 6 | **What stays single-tenant forever** vs. what becomes multi-tenant-capable in v2 — Modular Platform v2 primitives need a target list.    |
| 7 | **Cost target for hybrid cloud** — Cost audit r1 needs a number. What's "good"?                                                            |

---

## 10. Versioning

Spec versioning matches release cadence: every 6 months the spec re-bases off the v1/v2/v3 line that just shipped.

| Version                              | Date              | Trigger                                                                                          |
| :----------------------------------- | :---------------- | :----------------------------------------------------------------------------------------------- |
| **v1.0** (this version)              | 2026-05-13        | Initial; written pre-workshop.                                                                   |
| **v1.1** *(planned)*                 | 2026-05-18        | Post-workshop; engineer assignments + v1 scope sub-specs added.                                  |
| **v2.0** *(planned)*                 | 2026-10-01        | Post-EOQ3; what shipped, what slipped, v2 scopes for Q1 2027 sharpened.                          |
| **v3.0** *(planned)*                 | 2027-04-01        | Post-EOQ1 v2; replaced by a fresh 12-Month Deep Dive for Q3 '27 → Q2 '28.                        |

### Change log

- **2026-05-13** — v1.0 initial.

---

## 11. Promotion checklist (Jimmy)

When you approve this spec:

- [ ] Treat this spec as the workshop pre-read — circulate to attendees.
- [ ] *(Optional)* Pin a one-paragraph summary to [[Project Overview (JetX Tech Vision)]] under a new `## 12-Month Plan (v1.0)` heading; full doc stays here.
- [ ] *(Optional)* Create or update `Roadmap (JetX Tech Vision).md` from the matrix in §4.
- [ ] *(Optional)* Add a [[Decision Log (JetX Tech Vision)]] entry: `2026-05-13 — 2026 framed as "Year of Foundation"; charging/partners/MT-production deferred to Year 2.`
- [ ] After workshop: update this note with workshop outcomes; set `status: actioned`.
- [ ] Set [[[C] Proposal — Add 12-Month Deep Dive as Tangible Outcome]] `status: actioned` (it's no longer a proposal — the deep dive exists).

---

## 12. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project; this spec is its 12-Month Deep Dive sub-project output.
- [[[C] Vision and Mission — Design Spec]] — North Star v1.0; this 12-month plan operates under it.
- [[[C] Proposal — Add 12-Month Deep Dive as Tangible Outcome]] — the proposal that called for this sub-project.
- [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]] — strategic context the year arc operates under.
- [[Project Overview (FinOps)]], [[Project Overview (Identity)]], [[Project Overview (Jetx-Lakehouse)]], [[Project Overview (Jetx-Tickets)]] — in-flight projects whose milestones are encoded in this matrix.
- [[jetx-platform-principles]], [[jetx-data-ownership-principles]], [[00-architecture]] — the spine this plan is an instance of.
