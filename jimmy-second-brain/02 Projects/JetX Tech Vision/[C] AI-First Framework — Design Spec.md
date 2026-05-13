---
type: agent-input
project: JetX Tech Vision
author: claude
kind: proposal
status: open
date: 2026-05-13
sub-project: AI-First Framework
---

# AI-First Framework — Design Spec (v1.0)

> **Proposal.** How AI agents build software at JetX in a way that aligns to the vision, architecture, and tech stack — strictly. The framework operationalizes [[[C] Governing Principles — Design Spec]] Pillar E and gives both human fleet operators and AI agents (Claude, Laika, Nemo, George, Milu, Kelly, future agents) a shared rulebook. Once Jimmy approves, the framework promotes into [[Project Overview (JetX Tech Vision)]] and informs every new app, every new module, every agent-authored PR.

## 1. What this framework is — and isn't

**This framework IS:**

- A working model for how the AI-augmented JetX team builds software.
- The agent lifecycle: from spec → build → review → ship → iterate, with named gates at each step.
- The alignment surface: how agents align to [[jetx-platform-principles]] (Pillar B) and [[jetx-data-ownership-principles]] (Pillar C) without humans having to enforce every line of code.
- The role taxonomy: which agent roles exist, which JetX bots fill them, how new agents get onboarded.

**This framework is NOT:**

- A general AI engineering guide. Generic AI/ML best practices live elsewhere.
- A prompt library. Prompts live with the agents that use them.
- A tool / model selection guide. That changes too fast; we'll keep it as a separate living doc.
- A replacement for [[jetx-platform-principles]] or [[jetx-data-ownership-principles]]. Agents follow those; the framework just says *how*.

## 2. The working model — fleet operator + agent fleet

JetX engineering is not a team of solo coders. It's a team of **fleet operators**, each running a fleet of AI coding agents.

```
┌─────────────────────────────────────────────────────────┐
│                  Fleet Operator (human)                  │
│                                                          │
│  Job: steer the fleet. Choose what to build, in what    │
│  order, with which constraints. Hold architectural      │
│  alignment. Decide what ships.                          │
└─────────────────────────────────────────────────────────┘
              │ specs, reviews, decisions
              ▼
┌─────────────────────────────────────────────────────────┐
│                    Agent Fleet (AI)                      │
│                                                          │
│  Job: build. Translate specs to code. Run tests. Open   │
│  PRs. Iterate on review feedback. Operate at AI speed.  │
└─────────────────────────────────────────────────────────┘
              │ PRs, test results, telemetry
              ▼
┌─────────────────────────────────────────────────────────┐
│                  Live Production System                  │
└─────────────────────────────────────────────────────────┘
```

**Three division-of-labor rules:**

- **Steering belongs to the human.** Choosing what to build, in what order, with which trade-offs. Agents don't steer; agents execute.
- **Authoring belongs to the agent.** Translating a spec into code, running tests, opening PRs, iterating. Humans don't author production code by hand when an agent could.
- **Decisions belong to the human.** Merging, shipping, escalating, retracting. Agents don't decide; agents propose.

These rules are restated from [[[C] Governing Principles — Design Spec]] Pillar E (E6: agents build, humans steer).

## 3. The agent lifecycle — spec to ship

The standard work unit at JetX is a **module increment** (a unit of work that takes a module from one state to the next). Every increment goes through this loop:

```
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                 │
   │   1. SPEC          2. PLAN          3. BUILD         4. REVIEW  │
   │  (human leads)   (agent leads)    (agent leads)   (human leads) │
   │       │              │                │               │         │
   │       ▼              ▼                ▼               ▼         │
   │  What's the      Step-by-step    Code + tests +   Architectural │
   │  smallest        plan with       PR with          alignment,    │
   │  real thing?     dependencies    self-validation  correctness   │
   │                                                                 │
   │   5. SHIP          6. OBSERVE       7. ITERATE                  │
   │  (auto / human    (agent leads)   (agent + human, loop back     │
   │   gate)            metrics, logs,  to step 3 or 4)              │
   │       │              alerts          │                          │
   │       ▼              │                ▼                         │
   │  Flag-gated         ▼              Loop until v1 (then v2,      │
   │  prod deploy   Live-prod data       then v3) — build mindset    │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘
```

**Per-step responsibilities:**

| Step | Human role | Agent role | Output |
|---|---|---|---|
| 1. Spec | Author the spec; sharpen the "smallest real thing" | Read existing context; surface gaps; propose clarifying questions | Spec doc (or spec section, for small increments) |
| 2. Plan | Review the plan; approve before build starts | Decompose spec into ordered steps with named dependencies | Plan doc |
| 3. Build | Available for unblocking | Write code + tests; run local validation; open PR | PR |
| 4. Review | Architectural alignment, correctness, integration, business logic | Address review feedback; iterate the PR | Approved PR |
| 5. Ship | Gate the merge (or auto-merge if gates green) | N/A | Live in prod behind flag |
| 6. Observe | Watch first traffic | Collect metrics + logs; flag anomalies | Telemetry signal |
| 7. Iterate | Decide what next iteration solves | Loop back to step 3 or 4 | Next increment |

## 4. Quality gates — what makes a PR shippable

Three layers of gates: automated, agent-paired, human.

### 4.1 Automated gates (must be green before review)

- Lint + format pass
- Type check (or compile) pass
- Unit tests pass
- Integration smoke tests pass
- Security scan pass (dependency vulns, secret detection)
- Style/architectural lint (custom rules — see §5)

If any automated gate fails, the agent fixes it before requesting review. Humans don't review broken PRs.

### 4.2 Agent-paired gates (one agent reviews another's PR)

For PRs above a complexity threshold (e.g., >100 lines changed, or touching spine code), a *different* agent reviews before a human sees it. This catches:

- Architectural drift the authoring agent missed.
- Test coverage gaps.
- Duplication of existing utilities.
- Spec mismatch (what the PR does vs what the spec said).

Agent-paired review is checked but never *gating* — a human always sees the result and may override. (See [[[C] Governing Principles — Design Spec]] E5: review is the bottleneck; the agent-pair gate is a throughput multiplier, not a substitute for the human.)

### 4.3 Human gates (the merge button)

The fleet operator (or a peer fleet operator for spine code) reviews for:

- **Architectural alignment** — does this fit the spine? (Pillars B + C)
- **Business correctness** — does this do what the spec said *for the actual use case*?
- **Integration safety** — does this respect contracts to other modules?
- **Operational readiness** — is the rollout plan + flag strategy + monitoring adequate?

Human review SLA target: **<24 hours** from PR open to first review touch. If review is consistently >24h, the bottleneck is real and needs investment (more reviewers, better automation, tighter spec quality).

## 5. Architectural alignment — how agents stay on the spine

Agents will drift. The framework holds the line.

### 5.1 Spine-aware specs

Every spec MUST explicitly reference:

- The North Star term(s) it advances.
- The Pillar B principles it implicates (e.g., "this module is a new app — see Platform Principle #1: apps stay independent").
- The Pillar C channel position (source-of-truth / command writer / downstream sink).

If a spec doesn't trace upward, the agent flags it back to the human before building. (This is the spec-clarity leverage from Pillar E3.)

### 5.2 Architectural lint rules (custom)

Encode the spine-most-violated rules as automated checks:

- No cross-app database reads (Pillar B #5).
- No app-local user tables keyed by email (Pillar B #2).
- No direct DB writes outside the owning app (Pillar C #1).
- No sync-over-queue patterns (Pillar C #8).
- All change events use the CloudEvents envelope (Pillar C envelope spec).
- All new entity types declare their channel position in a header comment.

These run in CI. Failures block merge.

### 5.3 The drift report

Once per month, the framework owner runs a drift report:

- PRs merged this month that touched spine code.
- Architectural lint violations that were overridden.
- Modules whose channel positions changed.

The report goes to all fleet operators. Recurring drift becomes either a new lint rule (mechanize the enforcement) or an amended principle (admit the principle needs to change).

## 6. Common failure modes — and the override patterns

Agents fail in predictable ways. The framework names them so humans know what to look for.

| Failure mode | What it looks like | Override pattern |
|---|---|---|
| **Spec drift** | Agent produces code that solves a *similar* problem but not the actual spec | Sharper spec sentence ("the use case is X for user Y in context Z"); re-spec, re-run |
| **Architectural drift** | Code works locally; violates platform principle | Architectural lint catches most; human review for the rest; principle amended if recurring |
| **Test theater** | Agent generates tests that don't actually test the behavior | Human-reviewed test names; require one *failing-first* test per increment (TDD discipline for spine code only) |
| **Plumbing inflation** | Agent adds abstractions, factories, helpers not in the spec | Spec must explicitly call for abstractions; agent defaults to inline + obvious |
| **Stale-context refactor** | Agent renames or restructures unrelated code "while we're here" | PR scope must match spec scope; out-of-scope refactors get separate PRs |
| **False completion** | Agent reports "done" while edge cases or integration is missing | "Done when" checklist in the spec (observable items); CI verifies; review verifies |
| **Cargo-culted patterns** | Agent imports patterns from training data that conflict with the spine | Architectural lint + Pillar B/C principles surfaced in the spec |
| **Silent dependency add** | Agent adds an external library to solve something | Dependency review automated; new deps require human approval (Pillar B #10: boring infrastructure) |

## 7. Agent role taxonomy

JetX agents fill named roles. Multiple bots can fill the same role; one bot can fill multiple roles.

| Role | Job | JetX bots (current) |
|---|---|---|
| **Spec author** | Help humans write spec docs; surface gaps; propose clarifying questions | Claude (this doc), other agents as Jimmy designates |
| **Plan author** | Decompose specs into ordered plans with dependencies | (any spec-author agent) |
| **Builder** | Translate plan to code + tests + PR | Open question: which bots are primarily builders |
| **Reviewer (paired)** | Pre-review another agent's PR for architectural drift, test gaps, duplication | Open question; needs to be a *different* agent than the builder |
| **Ops monitor** | Watch metrics/logs/alerts in live prod; flag anomalies; trigger iteration | Open question; ties to Pillar D8 (bugs are routine) |
| **Spec librarian** | Maintain spec index, cross-references, alignment to North Star | This vault's [C] notes pattern; could be agent-automated in Y2 |
| **Principle-challenge agent** | Surface contradictions between principles + decisions; propose amendments | Future role |

> **Note:** Jimmy fills in the current bot → role mapping (Laika, Nemo, George, Milu, Kelly assignments). This spec defines the roles; bot assignments are operational.

## 8. Onboarding a new agent

Adding a new bot to the JetX fleet:

1. **Define the role.** Which of the roles in §7 does this bot fill? (Add a new role to §7 if needed.)
2. **Assign an agent code.** Two-letter or single-name prefix (e.g., `[C]`, `[Nemo]`). Used in input notes per CLAUDE.md's agent contribution protocol.
3. **Provide context.** Bot reads: this spec, [[[C] Governing Principles — Design Spec]], [[[C] Vision and Mission — Design Spec]], [[jetx-platform-principles]], [[jetx-data-ownership-principles]], [[00-architecture]], the relevant project's `Project Overview`.
4. **First task — low stakes.** Bot completes one non-spine increment with a fleet operator pairing. Output reviewed for spine alignment.
5. **Add to the agent inventory.** Update §7 mapping.
6. **30-day review.** Did the bot's PRs require excessive human override? Did it produce drift? Decide: keep, retrain prompts, retire.

## 9. AI domain headlines (per [[[C] Governing Principles — Design Spec]] E2)

Four AI domains, one headline per year:

| Year | Headline domain | What it means |
|---|---|---|
| **Y1 (2026)** | **Building software** | Team velocity. The agent fleet ships v1 across 9 net-new modules + 4 in-flight tracks by EOQ3 2026. Frame everything as "how does this make the team faster + more aligned?" |
| **Y2 (2027)** | **JetX's own ops** | Cheaper to run multi-service / multi-region. AI handles scheduling, demand forecasting, ops monitoring, anomaly detection. Frame: "the platform stays cheap as it grows." |
| **Y3 (2028)** | **Partner ops + Driver experience** | Twin headlines as the partner platform GAs and the CRM data asset gets deep enough for driver-facing AI. The other two domains continue in background. |

These are *headlines*, not exclusions — all four domains advance every year. The headline is what gets company-level prioritization, communication, and metrics.

## 10. Measuring AI-augmented team effectiveness

Metrics the framework owner watches:

| Metric | What it measures | Why it matters |
|---|---|---|
| **Increments shipped per fleet per quarter** | Throughput at the fleet-operator level | Pillar E1 economic claim |
| **Human review SLA (median + p95)** | The actual review bottleneck | Pillar E5 |
| **Architectural lint violation rate (per PR)** | Drift pressure | §5.2 |
| **PR rework count (avg revisions before merge)** | Spec clarity health | Pillar E3 |
| **Time from PR open to live prod** | End-to-end fleet velocity | Pillar D6 |
| **Live-prod incident rate per module v1 launch** | Quality of first ship | Pillar D8 |
| **Override rate (architectural lint exceptions / month)** | When principles need amendment | §5.3 |

Targets get set in [[[C] 12-Month Deep Dive — Design Spec]] (Q3 2026 baselines) and re-evaluated at the v2 line.

## 11. The framework's own roadmap

| Year | Framework state |
|---|---|
| **Y1 (2026)** | This spec v1.0. Agent lifecycle codified. Architectural lint rules drafted. Drift report monthly. Headline domain: Building software. |
| **Y2 (2027)** | v2 spec. Add: ops-monitor agent role in production; partner-ops AI domain ramping. Multi-region considerations (agents in different time zones / locales). |
| **Y3 (2028)** | v3 spec. Add: driver-facing AI domain operational; AI ethics + governance principles (data minimization, opt-outs, model behavior boundaries — referenced in [[[C] Governing Principles — Design Spec]] §12.5). |

## 12. Open questions

These don't block the spec but are flagged for follow-up:

1. **Current bot → role mapping.** Which JetX bots are primarily builders vs reviewers vs ops monitors? (§7)
2. **Architectural lint rules — concrete list.** Which patterns to mechanize first? (§5.2)
3. **Agent-paired review threshold.** What complexity triggers it? (§4.2)
4. **Drift report owner.** Who runs the monthly report? (§5.3)
5. **Live-production iteration cadence per module.** Each module has different real-world traffic; iteration cycle time varies. Need per-module calibration.
6. **Tool / model selection living doc.** This spec deliberately leaves it out; that doc needs its own home.
7. **External AI partnerships.** Does JetX use external AI APIs (OpenAI, Anthropic) or only self-hosted? Affects the partner-ops + driver-experience domains in Y2-Y3.

## 13. Versioning

- **v1.0 — 2026-05-13.** Initial; framework codified.
- **Revisit triggers:**
  - Y1 EOQ3 — what did the AI-augmented team actually ship? Recalibrate metrics + agent roles.
  - Y2 kickoff — add ops-monitor role to canon if Y1 proved it out.
  - When a new AI domain headline shifts (E2 in Pillar E).
- **Cadence:** annual review aligned with the 12-Month Deep Dive re-base.

### Change log

- 2026-05-13 — v1.0 initial.

## 14. Promotion checklist (Jimmy)

When you approve this spec:

- [ ] Add a `## AI-First Framework (v1.0)` section to [[Project Overview (JetX Tech Vision)]] referencing the §2 working model + §3 lifecycle; full doc stays here.
- [ ] Map current JetX bots (Laika, Nemo, George, Milu, Kelly) to §7 roles — update §7 inline or file a follow-up note.
- [ ] Decide on architectural lint rule priorities (§5.2 open question #2).
- [ ] Set this note's `status` to `actioned`.
- [ ] *(Optional)* Add a [[Decision Log (JetX Tech Vision)]] entry: `2026-05-13 — AI-First Framework v1.0 adopted; agent lifecycle, quality gates, and role taxonomy codified.`

## 15. See Also

- [[Project Overview (JetX Tech Vision)]] — parent project.
- [[[C] Vision and Mission — Design Spec]] — North Star v1.0 (AI is a load-bearing term).
- [[[C] Governing Principles — Design Spec]] — Pillar E (AI principles) operationalized here.
- [[[C] 12-Month Deep Dive — Design Spec]] — Y1 plan; framework's first proving ground.
- [[[C] 3-Year Roadmap — Design Spec]] — AI domain headlines per year.
- [[jetx-platform-principles]] — Pillar B; agents must align.
- [[jetx-data-ownership-principles]] — Pillar C; agents must align.
- [[00-architecture]] — the stack agents build on.
- [[CLAUDE]] — vault-level agent contribution protocol referenced in §8.
