---
type: agent-input
project: JetX Tech Vision
author: claude
kind: note
status: open
date: 2026-05-13
sub-project: 12-Month Deep Dive
audience: Jimmy (presenter)
purpose: slide-note bullets for high-level workshop presentation 2026-05-16/17
---

# 12-Month Workshop — Slide Notes

> [!info] How to use this
> Each section below = one slide. The **Title** is the slide heading. The **On-slide** bullets are what attendees see (keep these short and visual). The **Speaker notes** are what you say. Build your slides in your tool of choice; this is the script + structure.

---

## Slide 1 — Where we'll be at end of Q3 2026

**Title:** End of Q3 2026 — Everything is real.

**On-slide:**
- Every driver → CRM record, vehicle profile, loyalty balance, JetX Coins wallet
- Every site → wash + detail + vending in one catalog
- Every machine → telemetry through one IoT spine
- Every license plate at pilot sites → resolved to a known driver
- Platform → modular, contract-defined, multi-tenant-ready architecturally
- Spine (FinOps, Identity, Lakehouse, Tickets) → in daily production use

**Speaker notes:**
This is the destination. 4.5 months from this workshop. Not a vision document — a contract. By end of September, this list is true or we have a problem. The rest of the slides are how we get there.

---

## Slide 2 — Why 2026 is the "Year of Foundation"

**Title:** 2026 — Year of Foundation.

**On-slide:**
- We are NOT chasing new service types this year.
- We are making the platform able to *host* any service type cleanly.
- 2027+ layers differentiated features on a solid base.

**Speaker notes:**
Strategic call: we deferred charging, partner operators, and production multi-tenancy to Year 2. Reason — every Year-2 feature needs the foundation we're building this year. If we chase Year-2 features now, we either ship broken Year-2 features OR we never build the foundation. Both lose. The foundation is the bet.

---

## Slide 3 — Three pillars + the spine

**Title:** What we're building.

**On-slide:**

| Pillar                                  | Modules                                                                 |
| :-------------------------------------- | :---------------------------------------------------------------------- |
| **P1 — Modular Platform Architecture**  | Modular Platform, Multi-services, Vending, IoT, Vehicles Detection      |
| **P2 — Customer Relationship Layer**    | CRM, Loyalty, JetX Coins, Marketing tools                               |
| **P3 — Hybrid Cloud Hardening**         | Cost optimization, robustness, observability                            |
| **Spine (in-flight)**                   | FinOps, Identity, Lakehouse, Tickets                                    |

**Speaker notes:**
P1 is the platform itself — refactored to host any service. P2 is the driver relationship layer — owns identity, vehicle, loyalty, currency. P3 is the infra hardening — cost down, uptime up, observability on. The spine is already in flight from prior projects; each one graduates from "project" to "platform service" this year.

---

## Slide 4 — The cadence

**Title:** Every 6 months ships a major release.

**On-slide:**
- **EOQ3 2026** → v1 line — every module real and in production
- **EOQ1 2027** → v2 line — every module full scope
- **EOQ3 2027** → v3 line — differentiation features (out of window)

**Build philosophy:** parallel, not waterfall. **Build → test → fix → test → build more → improve. Loop in production.**

**Speaker notes:**
Two things to internalize. First — the 6-month rhythm. Major releases are predictable and frequent. Second — we don't wait for one module to finish before starting another. Everything builds in parallel. v1 isn't "finished"; it's the moment the loop starts running with real users. Polish is earned in iteration, not planned on a schedule.

---

## Slide 5 — What "v1" means (the contract)

**Title:** v1 = real, in production, one use case proven.

**On-slide:**
- ✓ Real — runs in live production
- ✓ One use case — proven end-to-end
- ✗ NOT feature-complete
- ✗ NOT polished
- ✗ NOT "good enough for everyone"

**Speaker notes:**
This is the contract that makes Q3 possible. If we let "v1" drift toward "polished" or "feature-complete," we hit zero v1s by end of Q3. With the discipline, we hit nine. Polish lives in v2. Edge cases live in v2. v1 is just: one real driver doing one real thing in real production, end-to-end, working.

---

## Slide 6 — The matrix (compressed)

**Title:** What ships in each quarter.

**On-slide:** *(show the §4 matrix from the spec — compact version, color-coded by pillar)*

**Speaker notes:**
Don't read the cells. Point at three things: (1) Q3 column is dense — that's the v1 line. (2) Q4 column is stabilization — fewer new things, more polish. (3) Q1 2027 column is dense again — v2 line. Direct attendees to the full spec for cell-by-cell detail.

---

## Slide 7 — Capacity — we are an AI-augmented team

**Title:** Why this is feasible.

**On-slide:**
- 8 engineer-fleets (not solo coders)
- Each fleet operator steers a fleet of AI coding agents
- Real constraints: spec clarity, code review, integration, deployment, **live-prod iteration**

**Throughput math:**

| Side     | Math                                       | Result          |
| :------- | :----------------------------------------- | :-------------- |
| Demand   | 9 net-new × 7 iterations                   | **63 needed**   |
| Supply   | 8 fleets × 50 iteration-slots (in 100 days) | **400 available** |
| Headroom | 63 ÷ 400                                   | **~6×**         |

**Speaker notes:**
The math is comfortable. But — raw throughput isn't the bottleneck. The bottleneck is **launch sequencing**: we can't take 9 modules to live prod the same week. Ops absorbs them sequentially. The hardest output of this workshop is the Q3 launch calendar.

---

## Slide 8 — What we're NOT doing

**Title:** Non-goals — intentional deferrals.

**On-slide:**
- Charging / EV launch → Year 2
- Partner operators → Year 2
- Multi-tenancy in production → Year 2
- Geographic expansion → defer
- Series A close → separate workstream
- Public brand work → separate workstream

**Speaker notes:**
These are real choices. Saying no to charging in 2026 is hard — IoT Platform v1 is happening, the temptation will be to "just plug a charger in." Don't. The discipline of NOT shipping charging in 2026 is what makes 2027 charging successful. Same logic across the list.

---

## Slide 9 — What this workshop must produce

**Title:** The 7 questions we need to answer.

**On-slide:**

| # | Question                                                              |
| :- | :-------------------------------------------------------------------- |
| 1 | Who carries which group of modules?                                  |
| 2 | What's each module's v1 scope (1 page each)?                          |
| 3 | What are review patterns for AI-augmented PR work?                    |
| 4 | What's the integration test strategy?                                 |
| 5 | What's the Q3 launch calendar?                                        |
| 6 | What goes into Modular Platform v2 multi-tenant primitives?          |
| 7 | What's the hybrid cloud cost target (audit r1)?                       |

**Speaker notes:**
Walk through the agenda — which questions land Day 1, which land Day 2, what's whole-room vs. breakout. See the Discussion Questions doc for the room-by-room format.

---

## Slide 10 — Close

**Title:** What the next 4.5 months feel like.

**On-slide:**
- Build, test, fix, test, build, improve. Repeat in production.
- Ship behind feature flags. Iterate in main.
- v1 by end of Q3. v2 by end of Q1.
- Polish is earned. Not planned.

**Speaker notes:**
This is how we work. If you find yourself waiting for something to be "finished" before shipping, you're doing it wrong. If you find yourself in a long-lived feature branch, you're doing it wrong. If you find yourself debating polish before v1 exists, you're doing it wrong. The team that builds JetX in 2026 is the team that ships into production every week and fixes in production every day.

---

## See also

- [[[C] 12-Month Deep Dive — Design Spec]] — the full spec
- [[[C] 12-Month Workshop — Executive Summary]] — attendee pre-read
- [[[C] 12-Month Workshop — Discussion Questions]] — workshop room agendas
