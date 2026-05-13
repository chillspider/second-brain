---
type: agent-input
project: JetX Tech Vision
author: claude
kind: note
status: open
date: 2026-05-13
sub-project: 12-Month Deep Dive
audience: workshop attendees (engineering + product)
purpose: pre-read for 2026-05-16/17 workshop
---

# 12-Month Plan — Executive Summary

> [!summary] One-paragraph version
> 2026 is the **"Year of Foundation."** We're not launching new service types this year — we're making the platform able to *host* any service type cleanly. By end of Q3 2026, every critical module ships v1 (real and in production for one use case). By end of Q1 2027, every module ships v2 (full scope). Build philosophy: parallel, not waterfall — **build → test → fix → test → build more → improve**, looping in production.

---

## What the year looks like

| Pillar                                  | Modules in this pillar                                                  |
| :-------------------------------------- | :---------------------------------------------------------------------- |
| **P1 — Modular Platform Architecture**  | Modular Platform, Multi-services, Vending Machines, IoT, Vehicles Detection |
| **P2 — Customer Relationship Layer**    | CRM, Loyalty, JetX Coins, Marketing tools                               |
| **P3 — Hybrid Cloud Hardening**         | Cost optimization, robustness, observability                            |
| **Spine (already in-flight)**           | FinOps, Identity, Lakehouse, Tickets                                    |

---

## The cadence

| Release line                              | What it means                                                                                  |
| :---------------------------------------- | :--------------------------------------------------------------------------------------------- |
| **EOQ3 2026 — v1 line** *(4.5 months out)* | Every critical module is real and in production for at least 1 real use case.                  |
| **EOQ1 2027 — v2 line**                   | Every module covers its full intended scope. Polish + scale + edge cases done.                |
| **EOQ3 2027 — v3 line** *(out of window)* | Differentiation layer — partner-readiness, charging, advanced AI features.                    |

> [!important] What "v1" means
> **v1 = the smallest thing that is real and in production for at least one real use case.** Not feature-complete. Not polished. Real + in production + one use case end-to-end. Polish lives in v2.

---

## The EOQ3 2026 "everything-is-real" moment

By end of Q3 2026, JetX is operationally a different company:

> [!success] What's true after Q3 2026
> - **Every driver** who interacts with JetX has a CRM record, vehicle profile, loyalty balance, and JetX Coins wallet.
> - **Every JetX site** sells more than wash — at least one detail service and vending products through one catalog.
> - **Every wash machine and vending machine** reports telemetry through one IoT spine.
> - **Every license plate** at pilot sites resolves to a known driver automatically.
> - **The platform** is module-shaped, contract-defined, multi-tenant-ready architecturally.
> - **The spine** (FinOps, Identity, Lakehouse, Tickets) is in daily production use.

---

## What we're NOT doing in 2026

> [!warning] Intentional deferrals — these are choices, not omissions
> - **Charging / EV service** — Year 2. IoT Platform v1 makes it possible later.
> - **Partner operator onboarding** — Year 2. Modular Platform v2 makes it possible later.
> - **Multi-tenancy in production** — Year 2. We build primitives only.
> - **Geographic expansion** — defer; competes with foundation work.
> - **Series A close** — separate workstream; FinOps reaches *ready* as a capability.
> - **Public brand work** (Wash24h vs. JetX) — separate workstream.

---

## Why this works — AI-augmented team

We're not human-throughput-bounded. Each engineer runs a fleet of AI coding agents. The 6-10 named engineers represent **fleet operators**, not solo coders.

**Real constraints (not raw code volume):** spec clarity, code review velocity, integration testing, deployment capacity, architectural coherence, and — the dominant one — **live-production iteration loops** (build → test → fix → loop in real prod, calendar-paced).

**Throughput math (back-of-envelope):**

| Side    | Calculation                                          | Result                                |
| :------ | :--------------------------------------------------- | :------------------------------------ |
| Demand  | 9 net-new modules × ~7 production iterations each    | **63 iteration-units needed**         |
| Supply  | 8 fleets × (100 days ÷ 2 days/iteration)             | **400 iteration-slots available**     |
| Headroom | 63 ÷ 400                                             | **~6× headroom on raw throughput**    |

The constraint isn't code volume — it's **launch sequencing and integration choreography**. That's what the workshop has to solve.

---

## What this workshop must produce

> [!question] Walking out of 2026-05-16/17, we should have answers to these

| # | Question                                                                                       | Workshop format                |
| :- | :--------------------------------------------------------------------------------------------- | :----------------------------- |
| 1 | Who carries which group of modules (engineer-fleet assignments)?                              | Whole-room, day 1             |
| 2 | What's each module's v1 scope, sharpened to 1 page?                                            | Pillar breakouts, day 1 PM    |
| 3 | What are our review patterns for AI-augmented PR work?                                         | Whole-room, day 2 AM          |
| 4 | What's the integration test strategy and who owns contract tests?                              | Pillar breakouts, day 2 AM    |
| 5 | What's the Q3 launch calendar — when does each module enter live prod?                         | Whole-room, day 2 PM          |
| 6 | What target list goes into Modular Platform v2 multi-tenant primitives?                        | Pillar 1 breakout             |
| 7 | What's the cost target for hybrid cloud (audit r1)?                                            | Pillar 3 breakout             |

---

## Homework before the workshop

Each attendee:

1. Read this summary + the full spec at [[[C] 12-Month Deep Dive — Design Spec]].
2. For modules you might own: pre-think the v1 scope. What's the smallest real thing in production?
3. For your area: pre-think the integration boundaries. Who needs what from you to ship their v1?

---

## See also

- [[[C] 12-Month Deep Dive — Design Spec]] — the full spec
- [[[C] 12-Month Workshop — Slide Notes]] — slide-note bullets for Jimmy's presentation
- [[[C] 12-Month Workshop — Discussion Questions]] — facilitator prompts per open question
- [[[C] Vision and Mission — Design Spec]] — North Star this plan operates under
- [[Project Overview (JetX Tech Vision)]] — parent project
