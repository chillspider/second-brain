---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-13
sub-project: Governing Principles
---

# Governing Principles — Design Spec (v1.0)

> **Proposal.** Single index + extension of all principles that govern JetX. Existing engineering docs ([[jetx-platform-principles]], [[jetx-data-ownership-principles]]) are imported wholesale and elevated to project-level canon. New principles fill the gaps: business, operations, AI, culture. Resolves Threads A (Trust) and B (B2B2C) deferred from [[[C] Vision and Mission — Design Spec]]. Once Jimmy approves, the pillar index promotes into [[Project Overview (JetX Tech Vision)]] and a follow-up `Principles Found (JetX Tech Vision).md` consolidates the canonical list.

## 1. What this is

A single index of every principle that governs JetX decisions — engineering, business, operational, AI, cultural. Not a re-write of existing rules. The existing engineering docs are **the canonical engineering layer**; this spec is the **layer above** that names *all* the principle domains, points to each domain's canonical source, fills the gaps, and defines the principle lifecycle (adopt / amend / retire).

## 2. The principle hierarchy

Three levels, top-to-bottom:

```
                ┌────────────────────────────────────────────┐
                │   LEVEL 1 — North Star (the sentence)      │
                │   "Own the high-frequency driver touchpoint │
                │    and route every other car-care service   │
                │    through one trusted, AI-powered platform"│
                └────────────────────────────────────────────┘
                                  │
                  ┌───────────────┴──────────────┐
                  ▼                              ▼
       ┌──────────────────────┐     ┌──────────────────────┐
       │ LEVEL 2 — Pillar     │     │ LEVEL 2 — Pillar     │
       │ principles           │     │ principles           │
       │ (5 pillars below)    │     │ ...                  │
       └──────────────────────┘     └──────────────────────┘
                  │                              │
                  ▼                              ▼
       ┌──────────────────────┐     ┌──────────────────────┐
       │ LEVEL 3 — Operational │     │ LEVEL 3 — Operational │
       │ rules (existing docs +│     │ rules                 │
       │ decision trees)       │     │                       │
       └──────────────────────┘     └──────────────────────┘
```

- **Level 1 — North Star.** One sentence. Stable. Versioned. Source: [[[C] Vision and Mission — Design Spec]].
- **Level 2 — Pillar principles.** ~3-7 principles per pillar. Plain language, decision-shaped. Owned here.
- **Level 3 — Operational rules.** Tactical, sometimes prescriptive (specific tools, schemas, decision trees). Owned in domain docs ([[jetx-platform-principles]], [[jetx-data-ownership-principles]], 12-month plan, etc.).

A principle at any level **must trace upward**. Operational rules must trace to a Level-2 pillar principle. Pillar principles must trace to the North Star. Anything that doesn't trace upward is either wrong or it's exposing a missing principle.

## 3. The five pillars

| Pillar | Domain | Canonical source |
|---|---|---|
| **A — Business principles** | Strategy, partner relations, what we sell, what we don't | This doc (§4) |
| **B — Engineering principles** | App boundaries, platform shape, integration patterns | [[jetx-platform-principles]] (imported) |
| **C — Data ownership principles** | Where data lives, how writes happen, how events flow | [[jetx-data-ownership-principles]] (imported) |
| **D — Operational principles** | How we build, ship, iterate, operate | This doc (§7) |
| **E — AI / Agent principles** | How AI agents operate, build software, scale the team | This doc (§8) |
| **F — Cultural principles** | How the team works, communicates, decides | This doc (§9) |

## 4. Pillar A — Business principles

Business principles govern *what we sell, who we sell to, and what we say no to*. They protect the North Star from market temptation.

**A1. The wedge is high-frequency; the margin is low-frequency.** We attract drivers through services they use weekly (wash, charging). We monetize through services they use rarely (detailing, oil, tires, partner services). Anything that breaks this funnel logic is a violation.

**A2. Build, don't buy.** We grow by building platform capability and inviting fragmented operators onto it. We don't acquire fragmented operators. Buying contradicts the "platform above operators" thesis — we become one more operator instead.

**A3. Trust is bi-directional and load-bearing.** Drivers trust the platform. Partners trust the platform. Drivers trust partner brands *through* the platform. The trust contract has three sides, not two. *Operationalized in §5 (Trust thread).*

**A4. Closed-loop currency.** JetX Coins is a closed-loop loyalty / value-transfer instrument. Not a crypto. Not publicly traded. Not transferable to third-party rails. The moment Coins becomes open-loop, we are a different company.

**A5. Acquire drivers through the touchpoint, not paid marketing.** The wedge IS the acquisition channel. If we find ourselves spending heavily on driver-acquisition marketing, the wedge isn't working — fix the wedge before spending more.

**A6. We are a platform, not a chain.** Once partners are onboarded (Y3+), our identity is the layer *above* operators. We never publicly position as a wash brand or an EV-charging brand; we are a platform that includes those services among others.

**A7. We don't manufacture hardware.** We integrate chargers, vending machines, sensors. We don't build them. The moment we manufacture, we take on a capital structure that doesn't fit the platform thesis.

**A8. Vietnam-first, regional-deliberate.** Vietnam is our base. Regional expansion is deliberate (Y2+ in the [[[C] 3-Year Roadmap — Design Spec]]), not opportunistic. We don't open a country because someone offers us a deal there.

## 5. Operationalizing Trust (Thread A resolution)

> *Deferred from V&M brain-dump Thread A.*

Trust is named in the North Star ("one *trusted*, AI-powered platform"). Operationally it's three contracts, not one:

| Trust contract | Who trusts whom | How we operationalize |
|---|---|---|
| **Driver → Platform** | Drivers trust JetX with identity, vehicle data, payment, loyalty balance, service experience | Data minimization; consistent service quality; transparent pricing; clear ownership of driver data; predictable loyalty rules |
| **Partner → Platform** | Partners trust JetX with customer flow, payment settlement, brand protection, technology | Transparent revenue share; predictable demand routing; SLA on tech reliability; clear data boundaries (we don't compete with our own partners) |
| **Driver → Partner brand (through platform)** | Drivers trust a JetX partner more than they'd trust that partner standalone | SOP enforcement (the partner pillar from V&M brain-dump §6.4); platform-enforced quality bar; rating + review system; platform-mediated dispute resolution |

**Principle (governing): Trust is built by what you *enforce*, not by what you *promise*.** SOPs, SLAs, dispute resolution, audit trails, transparent data flows — these are the trust scaffolding. Marketing language is not.

This principle constrains every Year 2 partner onboarding decision and every CRM / Marketing tools feature decision.

## 6. B2B2C structure (Thread B resolution)

> *Deferred from V&M brain-dump Thread B.*

JetX is operationally B2B2C: we serve **drivers** (C) directly through JetX-operated touchpoints, and we serve **partner operators** (B) who in turn serve drivers (B2B2C).

**Internal use:** We name the structure plainly. Teams understand which side they're building for. CRM team builds for drivers; Partner Operations team builds for partners; Marketing tools team builds for both.

**External use (the actual question):** Do we say "B2B2C" to outsiders?

- **To drivers:** No. Drivers see "JetX." The B2B2C structure is invisible.
- **To partners:** No, but we describe it directly — "we bring you customers, you provide service, we run the trust layer." The mechanics are explicit; the acronym isn't.
- **To investors:** Yes, on demand. The structure is a key part of the moat and revenue model.
- **In public-facing brand work:** Optional; defer to brand strategy (Year 2+ non-engineering workstream).

**Principle: Match the audience to the language.** Don't force B2B2C terminology into customer-facing copy; don't hide the structure from investors. Use it where it clarifies.

## 7. Pillar D — Operational principles

How we build, ship, iterate, operate. These come from the [[[C] 12-Month Deep Dive — Design Spec]] and apply across all years.

**D1. Build → test → fix → test → build more → improve. Loop in production.** Not waterfall. Each module is always in some part of the loop.

**D2. v1 = real, in production, for one real use case.** Not feature-complete. Not polished. Real users, real production, one use case proven end-to-end. Polish lives in v2.

**D3. Every 6 months ships a major release.** Predictable cadence: v1 EOQ3, v2 EOQ1, v3 EOQ3. Teams plan against this rhythm.

**D4. Parallel build, not waterfall.** All critical modules build in parallel toward each release line. We don't sequence modules.

**D5. Production is the integration environment.** Staging exists for safety gates, not for prolonged QA cycles. Feature flags + iterate-in-main.

**D6. No long-lived feature branches.** PRs merge fast; gates run automatic; flags hide unfinished work.

**D7. Polish is earned, not planned.** A module gets polish in the iteration that shows up needing polish — not on a schedule.

**D8. Bugs in production are routine, not exceptional.** Each module budgets for live-prod iteration loops as part of its v1 plan, not as overhead.

## 8. Pillar E — AI / Agent principles

How AI agents operate inside JetX. The team is AI-augmented; the product surface is increasingly AI-powered; the platform-ops layer leans on AI. These principles govern all three.

**E1. Tech is the moat. AI is the durable edge of that moat.** Fragmented competitors can copy services. They cannot easily copy a platform operated cheaply at scale by an AI-augmented team. Investment in AI capability is investment in the moat.

**E2. AI operates in four domains.** *(Resolution of Thread C from V&M brain-dump — all four, headlined in this order in 2026.)*
  1. **Building software** (team velocity) — Y1 focus, already operational.
  2. **JetX's own ops** (cheaper to run multi-service / multi-region) — Y1-Y2 ramp.
  3. **Partner ops** (workflows, scheduling, demand forecasting) — Y2-Y3 ramp.
  4. **Driver experience** (recommendations, predictive maintenance, concierge) — Y3 ramp.

Pick one *headline* per year for company-level focus; the others continue in the background.

**E3. Spec clarity is leverage.** Agents only build what specs describe. The team's output is bounded by spec quality, not by agent throughput. Time spent sharpening a spec is multiplied across every iteration the spec drives.

**E4. Humans hold architectural alignment.** Agents will drift toward what works locally; they don't enforce platform-level alignment. Humans review for alignment to the spine (Pillars B + C). Drift is normal and acceptable in isolated modules; drift in spine code is not.

**E5. Code review is the bottleneck, not authorship.** Optimize review patterns (paired review with agents, automated gates, batch review queues). Don't optimize raw code volume.

**E6. Agents build; humans steer.** The fleet operator's job is steering — choosing what to build, in what order, with which constraints. The agent's job is building. Don't conflate the two; don't have humans manually code when steering is the bottleneck.

**E7. Live-production iteration is calendar-paced, not AI-paced.** The dominant bottleneck (see [[[C] 12-Month Deep Dive — Design Spec]] §5 #6). AI doesn't speed up real-world iteration; it just frees humans to do more steering during the wait.

**E8. AI features are earned by data, not guessed.** Driver-facing AI features (E2 domain 4) launch only when the CRM data asset has enough depth to make them work. v1 wallet + identity is not enough; multi-year visit history is. This is why E2 domain 4 is Y3-not-Y1.

## 9. Pillar F — Cultural principles

How the team works together. Less codified than engineering, but no less governing.

**F1. Direct, terse, solution-oriented.** Internal communication leads with the ask or the answer. Padding gets stripped. The team values precision over politeness in technical discussion (and politeness in everything else).

**F2. CLI-first; pragmatic about tools.** Choose boring, established tooling. Avoid building bespoke abstractions. Anyone should be able to google any piece of the stack.

**F3. Tactical clarity over marketing language.** Internal docs (specs, deep-dives, principles) read like engineering docs, not marketing collateral. External docs are a separate workstream.

**F4. Output discipline.** This vault — and by extension this team — exists to produce things, not just store things. Every doc, every meeting, every decision should produce a tangible artifact or it doesn't happen.

**F5. Push toward output.** Reviews and discussions always end with "here's the next thing to make" — not "here's a summary of what we discussed."

**F6. Trust the team to ship.** Fleet operators are trusted to steer their fleets without micro-management. Trust is balanced by the v1-discipline rule (D2) and the production iteration loop (D1) — outcomes are visible quickly enough that mid-flight intervention is rarely needed.

**F7. Disagree by writing.** Disagreements that matter get written down (as proposals, decision-log entries, principle challenges). Verbal disagreements without artifacts evaporate.

## 10. How to use the principles

### As a decision filter

Every non-trivial decision (feature, hire, partner deal, market entry, architecture change) must:

1. **Trace upward** to a Level-2 pillar principle that supports it.
2. **Not contradict** any pillar principle, OR explicitly challenge the principle in writing.
3. **Be logged** in [[Decision Log (JetX Tech Vision)]] if it sets precedent.

If a decision traces to nothing, either the decision is wrong or a principle is missing — both deserve discussion.

### As a tie-breaker

When two paths are equally feasible, prefer the one that advances more pillar principles. Bonus weight if it advances principles from multiple pillars (e.g., a feature that advances both Business + Operational).

### As onboarding

Every new team member and every new partner reads §1-§3 + the index in §3. Domain specifics are read as needed.

### As a challenge surface

A principle that's been challenged and held is stronger than one nobody tested. Challenges go through §11 (principle lifecycle).

## 11. Principle lifecycle — adopt, amend, retire

**Adopt a new principle:**

1. File a `[<Agent>] Proposal — Principle: <title>.md` (or `[J] Proposal...` if Jimmy) in the JetX Tech Vision folder.
2. Argue: which pillar does it belong to, what gap does it fill, why isn't an existing principle enough.
3. Discuss + decide (Jimmy as final decider).
4. If adopted: add to this spec at the next minor version bump (v1.1, v1.2…); log in [[Decision Log (JetX Tech Vision)]].

**Amend an existing principle:**

- Wording-only tweak → patch bump (v1.0.1). No log entry.
- Meaning change → minor bump (v1.1). Log entry required.
- Principle invalidated or removed → major bump (v2.0). Log entry + cascading review of all sub-projects that referenced it.

**Retire a principle:**

- Only if it's *been challenged and held* multiple times, OR if business reality has fundamentally shifted (e.g., we pivot away from car care).
- Retirement is rare; principles should be edited or amended in place when possible.

## 12. Threads still open (for future spec versions)

These weren't resolved here. Carry forward.

1. **Series A timing** — does it warrant its own business principle? (e.g., "We don't structure technical decisions around funding milestones.") Pending decision in 3-Year Roadmap §9.
2. **Headcount trajectory principle** — at what point does the AI-augmented team stop scaling sub-linearly with footprint? (Pending data from Y2 ops.)
3. **Localization principles** — when we open the first non-Vietnam region, what does "one platform" mean for language, currency, regulation? Should produce Y2 principles.
4. **Open-source posture** — do we open-source any of our platform? (No principle exists; if relevant, propose one in Y2.)
5. **AI ethics / governance** — when E2 domain 4 (driver-facing AI) lands, principles about driver data + model behavior + opt-outs are needed. Defer to Y3 AI-First Framework.

## 13. Versioning

- **v1.0 — 2026-05-13.** Initial. Imports existing engineering docs; adds Pillars A, D, E, F; resolves Threads A and B from V&M.
- **Revisit triggers:**
  - Any principle is challenged in writing → discussion → possible amendment.
  - A new pillar emerges (e.g., regulatory/compliance becomes its own pillar in Y2+).
  - Every 6 months: light review for staleness alongside spec re-bases.

### Change log

- 2026-05-13 — v1.0 initial.

## 14. Promotion checklist (Jimmy)

When you approve this spec:

- [ ] Add a `## Governing Principles (v1.0)` section to [[Project Overview (JetX Tech Vision)]] with the §3 pillar table; full doc stays here.
- [ ] Populate `Principles Found (JetX Tech Vision).md` with the consolidated principle list (numbered, one line each, hyperlinked to this doc).
- [ ] *(Optional)* Add [[Decision Log (JetX Tech Vision)]] entries for the resolved threads:
  - `2026-05-13 — Trust operationalized as three contracts (driver→platform, partner→platform, driver→partner-through-platform); enforced via SOPs, SLAs, audit trails — not promises.`
  - `2026-05-13 — B2B2C terminology used internally + with investors; not in driver- or partner-facing copy.`
  - `2026-05-13 — AI operates in four domains; 2026 headline = building software; rest run in background.`
- [ ] Set this note's `status` to `actioned`.

## 15. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project.
- [[[C] Vision and Mission — Design Spec]] — North Star v1.0 (Level 1).
- [[[C] 12-Month Deep Dive — Design Spec]] — Level-3 operational rules for Y1 (Pillar D).
- [[[C] 3-Year Roadmap — Design Spec]] — Level-3 strategic sequencing.
- [[jetx-platform-principles]] — Pillar B canonical source.
- [[jetx-data-ownership-principles]] — Pillar C canonical source.
- [[00-architecture]] — stack-level view; Pillar B + C implementation.
- [[[C] Note — Vision and Mission Brain-Dump (Cowork Handoff)]] — origin of Threads A, B, C resolved here.
