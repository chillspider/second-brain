---
type: agent-input
project: JetX Tech Vision
author: claude
kind: note
status: open
date: 2026-05-13
sub-project: 12-Month Deep Dive
audience: workshop facilitators + breakout rooms
purpose: room-by-room agendas + facilitator prompts for 2026-05-16/17
---

# 12-Month Workshop — Discussion Questions

> [!info] How to use this
> Each section is a discrete workshop room. The **Goal** says what you're trying to walk out with. The **Prompts** are the conversation drivers. The **Artifact** is what gets written down and brought back to the whole-room readout.

---

## Suggested two-day agenda

| Day | Time   | Room                                                              | Format         |
| :-- | :----- | :---------------------------------------------------------------- | :------------- |
| 1   | AM     | **Whole-room** — Q1 (engineer-fleet assignments)                  | Open discussion |
| 1   | PM     | **Pillar breakouts** — Q2 (v1 scopes, one room per pillar)        | 3 parallel rooms |
| 1   | EOD    | **Whole-room** — readouts from pillar breakouts                  | Each pillar presents |
| 2   | AM     | **Whole-room** — Q3 (AI-augmented PR review patterns)             | Open discussion |
| 2   | AM     | **Pillar breakouts** — Q4 (integration test strategy)             | 3 parallel rooms |
| 2   | PM     | **Whole-room** — Q5 (Q3 launch calendar)                          | Calendar exercise |
| 2   | EOD    | **Whole-room** — Q6 + Q7 + close                                  | Open discussion |

---

## Q1 — Engineer-fleet assignments

**Goal:** Walk out with a named-person → module-group mapping.

**Format:** Whole-room, ~90 minutes, Day 1 AM.

> [!question] Prompts
> - Suggested mapping is in the spec §5 (Platform lead, Hardware lead, Relationship lead, Growth lead, etc.). Does the shape work? Where does it break?
> - For each engineer in the room: which fleet would you operate? Where do you have the most steering competence?
> - Who has prior context on which module? (Don't re-assign someone away from a module they've been live in.)
> - Anyone wearing two fleet-operator hats? (Acceptable if team is 6; problematic if team is 10.)
> - Backup operator for each fleet — single point of failure is unacceptable for any v1-by-Q3 module.

**Artifact:** A single page — `Engineer-fleet assignments (v1.0)` — with named primary + named backup for each of the 9 fleet groups.

---

## Q2 — v1 scopes per module

**Goal:** Walk out with a 1-page v1 scope spec for each of the 9 net-new modules.

**Format:** 3 parallel pillar breakouts, ~3 hours each, Day 1 PM. Each room covers its pillar's modules. Pillar 3 (Hybrid Cloud) doesn't have software v1s — it covers cost-audit r1 scope instead.

### P1 breakout — Modular Platform, Multi-services, Vending, IoT, Vehicles Detection

> [!question] Prompts (per module)
> - What's the **one** real use case that's proven end-to-end in v1? Name the use case in one sentence.
> - What's the **smallest** code/config surface that makes that use case work? Cut everything else.
> - What's the integration boundary with adjacent modules? (Multi-services ↔ Vending POS; IoT ↔ Vehicles Detection sensors; everything ↔ Modular Platform contracts.)
> - What's the **first** production deploy target — which 1-3 sites or device units?
> - What instrumentation is non-negotiable in v1 (so iteration loops work)?

**Artifact per module:** 1-page v1 scope spec. Same template across all 9:
- Use case sentence
- Code surface
- Integration boundaries
- First-deploy target
- Required instrumentation
- "Done when" checklist (~5 items, observable)

### P2 breakout — CRM, Loyalty, JetX Coins, Marketing tools

Same prompt template. Pay extra attention to the **data ownership** boundary:
- CRM is system of record for drivers + vehicles
- Loyalty is system of record for balances
- Coins is system of record for the currency ledger
- Marketing is a *reader* of CRM data + *writer* of campaign-execution events

Get these channel positions right in v1 or v2 becomes a rewrite.

### P3 breakout — Cost audit r1 + first cuts

> [!question] Prompts
> - What's our current cloud spend baseline by category (compute, storage, egress, managed services)?
> - Where are the easiest 20% cuts? (reserved capacity, idle workload shutdown, image right-sizing, log volume)
> - What's the target % reduction by EOQ3? (Need a number — see Q7.)
> - Which cuts are "safe" (no service impact) vs. "risky" (might affect performance)?
> - Who's on point for the cost audit?

**Artifact:** Cost audit r1 plan — categories, targets, owners.

---

## Q3 — AI-augmented PR review patterns

**Goal:** Walk out with the team's standard for reviewing agent-authored code.

**Format:** Whole-room, ~90 minutes, Day 2 AM.

> [!question] Prompts
> - When an agent submits a PR, what's the human reviewer's job? (correctness? architectural alignment? performance? all of the above?)
> - Should we require paired-review (two humans) for any class of changes? Which class?
> - What automated gates are non-negotiable? (lint, type, unit tests, integration smoke, security scan, dependency audit)
> - At what PR-rate do we need to invest in better review tooling? (Throughput math: ~8 fleets × N PRs/day each = ?)
> - What's the review SLA — how long can a PR sit before it blocks the next iteration?
> - Where do we *trust* the agent and where do we *verify*?

**Artifact:** `AI-augmented PR review playbook (v1.0)` — checklist of automated gates + human-review responsibilities + SLA.

---

## Q4 — Integration test strategy

**Goal:** Walk out with a named owner + plan for contract tests, ahead of Modular Platform v1.

**Format:** 3 parallel pillar breakouts (each pillar names its contract boundaries), ~2 hours each, Day 2 AM.

> [!question] Prompts (per pillar)
> - What are the contract boundaries this pillar exposes to other pillars/modules?
> - Who owns each contract — provider or consumer?
> - What does a contract test look like for our stack (.NET 9 + SvelteKit + PG17)?
> - When does each contract test get written — before the module ships v1, or as part of v1?
> - Where do contract tests run — CI? Live prod smoke? Both?
> - What's the alarm when a contract breaks?

**Artifact per pillar:** Contract inventory + test plan. Roll up into a whole-room contract registry.

---

## Q5 — Q3 launch calendar

**Goal:** Walk out with a week-by-week launch sequence for the 9 net-new v1s.

**Format:** Whole-room calendar exercise, ~2 hours, Day 2 PM. **The hardest output of the workshop.**

> [!warning] The constraint
> Ops can absorb roughly **one net-new module entering live prod per ~1.5 weeks**. Q3 is 13 weeks. 9 modules + 4 in-flight milestones = launch capacity is the limiting factor, not engineering throughput.

> [!question] Prompts
> - Order modules by dependency (Modular Platform v1 unlocks others; CRM v1 unlocks Loyalty/Coins/Marketing).
> - For each module, what's the *earliest* it can enter live prod? The *latest* before EOQ3?
> - Which two modules can launch the same week safely (independent integration surfaces)?
> - What's the "go-no-go" gate per launch — what observable signals must be green?
> - Who's on launch-day duty for each module?
> - When does the next-day post-launch review happen?

**Suggested calendar template:**

| Week | Module entering live prod | Launch lead | Go/no-go criteria | Post-launch review |
| :--- | :------------------------ | :---------- | :---------------- | :----------------- |
| W1   |                           |             |                   |                    |
| W2   |                           |             |                   |                    |
| ...  | (13 weeks)                |             |                   |                    |

**Artifact:** Filled 13-week launch calendar. This is the workshop's headline output.

---

## Q6 — Modular Platform v2 multi-tenant primitives target list

**Goal:** Walk out with the target capabilities for the v2 (EOQ1 2027) MT primitives.

**Format:** Pillar 1 breakout, ~90 minutes, Day 2 EOD.

> [!question] Prompts
> - What does "multi-tenant-ready" mean operationally? (config isolation? data isolation? RBAC scoping? all three?)
> - What MT primitives must be in v2 for Year 2 partner onboarding to be a config change, not a code change?
> - What can stay single-tenant in 2026/2027 even if MT primitives exist?
> - Where's the riskiest MT design decision — data sharding? identity scoping? billing isolation?
> - What 1-tenant + 1-fake-tenant test scenario validates v2?

**Artifact:** MT primitives target list — capability checklist, with which v2 module enables each capability.

---

## Q7 — Hybrid cloud cost target

**Goal:** Walk out with a number.

**Format:** Pillar 3 breakout (or whole-room), ~30 minutes, Day 2 EOD.

> [!question] Prompts
> - What's the 2026 baseline spend?
> - What's our gut % cut for r1? r2?
> - What's the absolute floor we'd never go below (capacity headroom + reliability)?
> - Does the cost target gate any other decision in 2026? (e.g., do we delay a launch to absorb a cost spike?)

**Artifact:** Two numbers — r1 target (by EOQ3) and r2 target (by EOQ1 2027), in % reduction from baseline.

---

## After the workshop

> [!success] What gets updated when we walk out
> - Update [[[C] 12-Month Deep Dive — Design Spec]] to v1.1 with all workshop outputs incorporated.
> - Engineer-fleet assignments → spec §5
> - v1 scope sub-specs → linked from spec §4 cells (one file per module)
> - PR review playbook → new file [[[C] AI-augmented PR review playbook]]
> - Contract registry → new file [[[C] Contract registry (v1.0)]]
> - Q3 launch calendar → new file [[[C] Q3 2026 Launch Calendar]]
> - MT primitives target list → linked from spec §4 Modular Platform v2 cell
> - Cost targets → spec §4 Hybrid Cloud Hardening cells
> - Set [[[C] 12-Month Deep Dive — Design Spec]] `status: actioned` after v1.1 update.

---

## See also

- [[[C] 12-Month Deep Dive — Design Spec]] — the full spec
- [[[C] 12-Month Workshop — Executive Summary]] — attendee pre-read
- [[[C] 12-Month Workshop — Slide Notes]] — Jimmy's slide-note bullets
