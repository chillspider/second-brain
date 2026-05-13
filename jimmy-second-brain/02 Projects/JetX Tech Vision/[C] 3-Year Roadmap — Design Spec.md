---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-13
sub-project: 3-Year Roadmap
---

# 3-Year Roadmap — Design Spec (v1.0)

> **Proposal.** Three-year directional plan, **Q3 2026 → Q2 2029**. Year 1 is the [[[C] 12-Month Deep Dive — Design Spec]] verbatim. Years 2 and 3 are directional, not tactical — quarter-level granularity for Y2, half-year for Y3. Once Jimmy approves, the year arcs promote into [[Project Overview (JetX Tech Vision)]] under a new `## 3-Year Roadmap (v1.0)` heading.

## 1. The three-year arc — one sentence per year

- **Year 1 — Year of Foundation** (Q3 2026 → Q2 2027). Build the platform right so future-year features land cleanly. See [[[C] 12-Month Deep Dive — Design Spec]].
- **Year 2 — Year of Service + Regional Kickoff** (Q3 2027 → Q2 2028). Operationalize the deferred services (detailing at scale, oil, tire, battery) and open the first new region.
- **Year 3 — Year of Wedge + Platform** (Q3 2028 → Q2 2029). Add charging (the second high-frequency touchpoint), onboard first partner operators in production, multi-tenant production GA, deepen the new region.

## 2. North Star tag

The roadmap is the operational expression of the [[[C] Vision and Mission — Design Spec]] North Star v1.0:

> *Own the high-frequency driver touchpoint, and route every other car-care service through one trusted, AI-powered platform.*

| Year | Advances which load-bearing terms |
|---|---|
| Y1 | **platform** (modular), **trusted** (CRM/Loyalty/Coins), **AI-powered** (team + spine) |
| Y2 | **route every other car-care service** (multi-services in production), **platform** (regional expansion proves portability) |
| Y3 | **high-frequency driver touchpoint** (charging adds 2nd touchpoint), **route every other... through one trusted platform** (partner operators live, MT production) |

By end of Y3, every load-bearing term in the North Star sentence is operationally true.

## 3. Year 1 — Year of Foundation (Q3 2026 → Q2 2027)

See [[[C] 12-Month Deep Dive — Design Spec]] for the full matrix. One-line summary per pillar:

- **P1 — Modular Platform Architecture.** Refactor wash codebase onto an expandable platform; host Multi-services, Vending, IoT, Vehicles Detection as v1 PoCs by EOQ3 2026.
- **P2 — Customer Relationship Layer.** CRM, Loyalty, JetX Coins, Marketing tools v1 by EOQ3 2026. Driver relationship becomes a system of record.
- **P3 — Hybrid Cloud Hardening.** Cost down, robustness up, observability on. Spine production-grade by EOQ1 2027.
- **Spine (in-flight).** FinOps, Identity, Lakehouse, Tickets graduate from project to platform service.

**End-of-Y1 state:** every driver has a CRM record + vehicle profile + loyalty balance + Coins wallet. Every JetX site sells more than wash. Platform is modular and MT-architecturally-ready (not yet MT-productized). Spine is in daily production use.

## 4. Year 2 — Year of Service + Regional Kickoff (Q3 2027 → Q2 2028)

### 4.1 Y2 thesis

Y1 left us with a modular platform that can host any service and a service catalog with stubs for detailing/oil/tire/battery. Y2 **operationalizes** those services and proves the platform portable by opening the first new region.

### 4.2 Y2 pillars

| Pillar | What's in it |
|---|---|
| **Y2-P1 — Service Operationalization** | Move detailing from wash-detail-variant to a real workflow (booking, technicians, service bays, parts). Launch oil change service. Launch tire service. Launch battery service. Each gets v1 (one site, one use case) then scales. |
| **Y2-P2 — Regional Kickoff** | First non-current-footprint region. **TBD: another Vietnam city, or first ASEAN city** (KL / Bangkok / Manila — Jimmy decides). 2-3 stations live by EOY2. |
| **Y2-P3 — Multi-tenant Productization** | Modular Platform v3 turns MT-readiness (Y1 v2 primitives) into MT-production. First non-JetX tenant runs on the platform — pilot, not GA. Sets up Y3 partner onboarding at scale. |
| **Y2-P4 — Platform Operations Maturity** | SLOs/runbooks/on-call (Y1 EOQ output) get exercised at 2× footprint. Hybrid cloud expands to new region's infrastructure. AI-first framework v2 — agents now build cross-service workflows. |

### 4.3 Y2 matrix (half-year granularity)

| Module / Track | H1 Y2 (Q3-Q4 '27) | H2 Y2 (Q1-Q2 '28) |
|---|---|---|
| **Detailing service** | v1: 1 station, 1 detail package, booking flow | Scale to 5 stations + 3 detail packages |
| **Oil change service** | v1 design + 1 station pilot | v1 production at 3 stations |
| **Tire service** | v1 design | v1 pilot at 1 station |
| **Battery service** | v1 design | v1 pilot at 1 station |
| **Regional expansion** | Region selected, 1st station opens | 2nd + 3rd stations open; localization (language, currency, payment) shipped |
| **Modular Platform v3 (MT-prod)** | MT plumbing wired; pilot 2nd tenant onboarded (internal/fake tenant) | First real external pilot tenant on the platform |
| **CRM/Loyalty/Coins evolution** | Cross-region loyalty rules; Coins multi-currency support | Cross-tenant identity scoping (drivers can use one identity across tenants) |
| **AI-first framework v2** | Agent fleets operating cross-service; first AI-built service workflow | Agent fleets operating cross-region |
| **Hybrid Cloud** | New region infrastructure; multi-region cost model | Multi-region observability; cross-region DR drills |

### 4.4 Y2 end-state

- 4+ services live and producing revenue (wash, detailing, oil, tire, battery, vending — pick the ones that hit v1 prod).
- 2-3 stations in new region.
- First non-JetX tenant on the platform (pilot, not partner-at-scale).
- Platform proven portable across regions and across tenants.

## 5. Year 3 — Year of Wedge + Platform (Q3 2028 → Q2 2029)

### 5.1 Y3 thesis

Y2 proved we can add services and add regions. Y3 closes the North Star: add the second high-frequency touchpoint (charging) and turn the platform into a real partner marketplace (multi-tenant at scale).

### 5.2 Y3 pillars

| Pillar | What's in it |
|---|---|
| **Y3-P1 — Charging Launch** | EV charging service goes live. IoT Platform v3 supports chargers + payment + load balancing. First 5 charger locations. |
| **Y3-P2 — Partner Operators GA** | First 10-50 partner operators onboarded onto the multi-tenant platform. Marketplace surface live. SOPs enforced. Driver-facing partner discovery. |
| **Y3-P3 — Regional Deepening** | 5+ stations in the new region(s); possibly open 2nd new region. Multi-region AI/ops fully balanced. |
| **Y3-P4 — Differentiation Layer** | Advanced AI features: predictive maintenance, demand forecasting, dynamic pricing, concierge driver experiences. The AI moat goes from operational ("makes the team fast") to product ("makes JetX a better service for the driver"). |

### 5.3 Y3 milestones (year-level, not quarter-level)

- **Charging:** 5+ locations live; integrated with wash + Coins + Loyalty.
- **Partners:** 10-50 partner operators across at least 3 service categories.
- **MT-production:** GA, not pilot. Onboarding a new tenant is a config change, not a code change.
- **Geographic:** 5+ stations in region 2; possibly stations in region 3.
- **AI-first:** at least 1 driver-facing AI feature shipped in production.
- **Series A:** closed if not already; revenue base reflects Y2/Y3 service + partner additions.

### 5.4 Y3 end-state — every North Star term is operationally true

- ✓ **Own the high-frequency driver touchpoint** — wash + charging both live.
- ✓ **route every other car-care service** — detailing/oil/tire/battery in production; partner services in production.
- ✓ **through one** — same identity, same coins, same loyalty across all services + tenants + regions.
- ✓ **trusted** — SOP enforcement live; partner brands benefit from JetX trust halo; driver-side trust evidenced by retention metrics.
- ✓ **AI-powered** — team built it cheap with AI agents; product has at least one driver-facing AI feature; platform ops use AI.
- ✓ **platform** — modular, multi-tenant, multi-service, multi-region. Not a chain. Not an aggregator.

## 6. Beyond Y3 — directional notes (not in this roadmap)

These aren't planned in this doc, but the roadmap creates the optionality for them:

- Regional scale beyond ASEAN
- Adjacent vertical extensions (fleet management, motorcycle/truck verticals)
- White-label platform offering (JetX-as-a-Service for other operators)
- Direct OEM integrations (car manufacturer partnerships)
- Driver financial products (credit, insurance) leveraging the CRM + Coins data asset

## 7. Cross-year dependencies

| Dependency | Year produces it | Year consumes it |
|---|---|---|
| Modular Platform v1 contracts | Y1 EOQ3 | Y2 service launches need stable contracts |
| Modular Platform v2 MT primitives | Y1 EOQ1 '27 | Y2 P3 MT-productization |
| CRM v2 (segments + journeys) | Y1 EOQ1 '27 | Y2 cross-region loyalty + Y3 driver-facing AI features |
| IoT Platform v2 (OTA, edge processing) | Y1 EOQ1 '27 | Y3 P1 charging launch |
| AI-first framework v1 (team velocity) | Y1 continuous | Y2 P4 cross-service agents, Y3 P4 driver-facing AI |
| Hybrid cloud SLOs + runbooks | Y1 EOQ2 '27 | Y2 P4 multi-region ops |

If any Y1 dependency slips, the corresponding Y2/Y3 milestone slips with it. This is why Y1 v1-discipline matters beyond Y1.

## 8. Non-goals across all 3 years

> What we explicitly will NOT do in this 3-year window:

- **Acquisitions.** Build, don't buy. Buying fragmented operators undermines the "platform above operators" thesis.
- **Hardware manufacturing.** We integrate chargers and vending machines; we don't build them.
- **Crypto / public token.** JetX Coins stays closed-loop. No tokenization, no public exchange listing.
- **Becoming a financial services company.** Driver financial products (insurance, credit) are post-Y3 if at all.
- **Becoming a software vendor.** White-label JetX is post-Y3 if at all.
- **Mass driver-acquisition spend.** We acquire drivers through the high-frequency touchpoint, not through paid marketing arms races.

## 9. Open questions

These can land outside this spec — captured for follow-up:

1. **Y2 region selection.** Another Vietnam city, or first ASEAN city? Decision needed by EOQ1 2027 to start regulatory + real-estate prep.
2. **Y2 service order.** Detailing first (highest fit with existing wash sites) or oil change first (highest frequency after wash)?
3. **Y3 charger sourcing.** Build proprietary charger hardware? Integrate 3rd-party (ABB, Schneider, local Vietnam brands)?
4. **Y3 partner economics.** Revenue share % per service? Subscription + take-rate? Flat fee?
5. **Series A timing.** Y1 EOQ produces "Series-A-ready" FinOps capability. When does the actual close get scheduled? Affects Y2 hiring + spending plan.
6. **Headcount trajectory.** Y2 + Y3 capacity model assumes the AI-augmented team scales sub-linearly with footprint. Need actual projection of fleet operators + total headcount per year.

## 10. Versioning

- **v1.0 — 2026-05-13.** Initial; written alongside the 12-Month Deep Dive.
- **Revisit triggers:**
  - Y1 EOQ ('26-09-30) — does what shipped change the Y2 thesis?
  - Y1 EOQ1 '27 ('27-03-31) — Y2 region + service order locked.
  - End of each year — replace Y-N with the next year's deep dive; roll the window forward.

### Change log

- 2026-05-13 — v1.0 initial.

## 11. Promotion checklist (Jimmy)

When you approve:

- [ ] Add a `## 3-Year Roadmap (v1.0)` section to [[Project Overview (JetX Tech Vision)]] with the §1 one-sentence-per-year summary; full doc stays here.
- [ ] *(Optional)* Add a [[Decision Log (JetX Tech Vision)]] entry: `2026-05-13 — 3-Year arc locked: Y1 Foundation / Y2 Service+Regional / Y3 Wedge+Platform.`
- [ ] Set this note's `status` to `actioned`.
- [ ] Decide Y2 region + Y2 service order by EOQ1 2027 (open questions §9.1, §9.2).

## 12. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project.
- [[[C] Vision and Mission — Design Spec]] — North Star v1.0; this roadmap operationalizes it.
- [[[C] 12-Month Deep Dive — Design Spec]] — Year 1 detailed plan (matrix + cadence + capacity).
- [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]] — strategic thesis the roadmap operates under.
- [[jetx-platform-principles]], [[jetx-data-ownership-principles]], [[00-architecture]] — the spine.
