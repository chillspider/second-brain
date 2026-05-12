---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-12
sub-project: Vision & Mission
---

# Vision & Mission — Design Spec (v1.0)

> **Proposal.** This file proposes the JetX North Star sentence and the rules for using it. Once Jimmy approves, he promotes the North Star into [[Project Overview (JetX Tech Vision)]] and marks this note `status: actioned`.

## 1. The North Star (v1.0)

> **Own the high-frequency driver touchpoint, and route every other car-care service through one trusted, AI-powered platform.**

One sentence. No formal Mission/Vision split (per the brainstorming decision — see [[#6 Provenance]] below). The present-vs-future distinction surfaced in the brain-dump lives across the downstream sub-projects (#2–#7) rather than being structurally split here.

## 2. Term decoder

Every word in the sentence is load-bearing. The decoder makes that explicit so future readers can't quietly drift away from the intent.

| Term | What it means | Why it's in the sentence |
|---|---|---|
| **Own** | Direct operation, not aggregation. JetX runs the customer relationship; we don't broker it. | Aggregators get squeezed; operators with the touchpoint don't. |
| **high-frequency driver touchpoint** | Wash + charging. Weekly or more often. | The wedge. Low-frequency car-care services can't earn the driver relationship on their own — that's why the industry stays fragmented. |
| **route every other car-care service** | Drive demand from the high-frequency top of funnel (wash, charging) into the low-frequency, higher-margin services (detailing, oil, tires, battery, etc.) | The funnel. This is how the touchpoint monetizes. |
| **one** | Multi-service, multi-tenant, multi-region — but ONE platform, ONE identity, ONE trust contract. | Without "one" it's a chain. With "one" it's a platform. |
| **trusted** | Bi-directional. Drivers trust the platform. Partners trust the platform. Drivers trust partner brands *through* the platform. | The two-sided trust is the moat against fragmentation. Without it, the partner pillar collapses. |
| **AI-powered** | Tech is the moat. AI/automation runs the platform cheaply at scale AND gives partners leverage they can't build alone. | This is what makes JetX defensible against larger, non-tech-native consolidators. |
| **platform** | Not a chain, not a marketplace, not an aggregator. A coordinated tech-and-operations layer above fragmented operators. | Names the company shape correctly. |

## 3. How it's used

The North Star is a decision tool, not a slogan.

- **Decision filter** — every proposal (feature, hire, partner deal, market entry) must tighten the touchpoint, deepen the routing, increase trust, or extend the AI moat. Proposals that do none of those four are suspect; document the justification anyway, or kill the proposal.
- **Tie-breaker** — when two paths are equally feasible, prefer the one that advances at least two of the sentence's load-bearing terms (touchpoint, routing, "one", trust, AI moat).
- **Onboarding** — every new team member and every new partner reads this first. It's the shortest possible answer to "what is JetX?"

## 4. What the North Star deliberately leaves OUT

Specific decisions on each of these are deferred to the matching sub-project in the queue. The North Star anchors them; it doesn't replace them.

| Deferred to | What's deferred |
|---|---|
| **Sub-project #2 — 12-Month Deep Dive** *(workshop, urgent)* | Quarter-by-quarter tactical plan for the next year |
| **Sub-project #3 — Platform & Stack Reference** | The actual tech stack |
| **Sub-project #4 — Architectural Structure** | How the system is organized |
| **Sub-project #5 — Governing Principles** | The full principles system + values + culture |
| **Sub-project #6 — AI-First Framework** | How AI agents build software aligned to the spine |
| **Sub-project #7 — 3-Year Roadmap** | Year 2 and Year 3 directional sketch; the 12-month plan feeds this |

Each downstream sub-project must show, in its own spec, how its decisions track the North Star.

## 5. Versioning

The North Star is supposed to be stable. Lightweight versioning catches drift.

- **v1.0 — 2026-05-12.** Initial.
- **Revisit triggers:**
  - A sub-project's tangible output contradicts a term in the sentence.
  - A major business milestone shifts the thesis: first non-Vietnam region, first 100 partner locations, first non-wash/charging high-frequency service launched.
- **Change discipline:**
  - Wording tweak → patch bump (v1.1, v1.2). Note in change log below; no Decision Log entry required.
  - A term changes meaning → minor bump (v2.0). [[Decision Log (JetX Tech Vision)]] entry required.
  - A term added or removed → major version (v3.0). [[Decision Log (JetX Tech Vision)]] entry + cascading review of all downstream sub-projects.

### Change log

- 2026-05-12 — v1.0 initial.

## 6. Provenance

- **Origin:** brainstorming session via `superpowers:brainstorming`, 2026-05-12.
- **Source material:** [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]]
- **Decision sequence (during brainstorming):**
  - **Q1:** structure → single North Star one-liner (over Mission+Vision pair, Mission+Vision+Values trio, or one combined paragraph).
  - **Q2:** AI/tech in or out of the sentence → IN. Jimmy: *"tech is our moat"*; the dating risk of the term "AI" is accepted.
  - **Q3:** which candidate sentence → high-frequency wedge made explicit (over the tighter baseline and the partner-named variant).
- **Threads still open (resolve via downstream sub-projects, not here):**
  - **Thread A — Trust:** how to make bi-directional trust operational. Resolves in Governing Principles (#5).
  - **Thread B — B2B2C structure:** whether/how to name the two-sided structure publicly. Resolves in Platform & Stack Reference (#3) or Governing Principles (#5).
  - **Thread C — AI applications:** which of the four AI uses (JetX ops / partner ops / driver UX / building software) gets primary investment first. Resolves in AI-First Framework (#6) and 12-Month Deep Dive (#2).

## 7. Promotion checklist (Jimmy)

When you approve this spec:

- [ ] Add the North Star sentence to the top of [[Project Overview (JetX Tech Vision)]] under a new `## North Star (v1.0)` heading, above `## Goal`.
- [ ] *(Optional, your call)* Pin a short callout to other active projects' Project Overviews so the North Star is visible to anyone working in [[Project Overview (FinOps)]], [[Project Overview (Identity)]], [[Project Overview (Jetx-Lakehouse)]], [[Project Overview (Jetx-Tickets)]].
- [ ] *(Optional)* Create [[Decision Log (JetX Tech Vision)]] if it doesn't exist yet and log: `2026-05-12 — North Star v1.0 adopted.`
- [ ] Set this note's `status` to `actioned`.
- [ ] Set [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]] `status` to `actioned`.

## 8. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project; this spec proposes content for it.
- [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]] — full thesis the North Star compresses.
- [[jetx-platform-principles]] — existing platform principles doc; reconciled in Governing Principles sub-project (#5).
- [[jetx-data-ownership-principles]] — same.
- [[00-architecture]] — current stack snapshot.
