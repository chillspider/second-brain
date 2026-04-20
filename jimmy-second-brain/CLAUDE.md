# Jimmy's Second Brain — Claude Context File

This file tells you everything you need to orient yourself in this vault. Read it at the start of every session.

**First stop for browsing:** [[[C] Vault Map]] — single-page MOC with one hop to anything important.

---

## Who This Vault Belongs To

Jimmy. Hands-on technical founder running JetX/Wash24h, an automated car wash operation with 50+ locations across Vietnam out of Ho Chi Minh City. Moves fluidly between business strategy and deep infrastructure work — architects FinOps platforms, hardens Tailscale ACLs, tunes PostgreSQL boxes, runs a multi-bot AI fleet, and sizes solar + BESS systems, often in the same week. Direct, terse, solution-oriented; CLI-first; pragmatic about cost and tool choice. Builds durable infrastructure at the edge of what a small team can manage, and refuses to let the business or the servers stay fragile.

**The governing principle:** This vault exists to produce things, not just store things.

---

## The Note Types (Front Matter: `type:`)

| Type | What It Is |
|---|---|
| `note` | Raw captures from external content (videos, books, pods) |
| `idea` | Something applicable to a current project RIGHT NOW |
| `solution` | Jimmy's worked answer to a specific problem — first principles, what's been tested, what's still open |
| `problems` | Project overview — goal, why, tangible outcomes, open problems. Read this first when working on any project. |
| `daily` | Daily log — what was worked on, what's still unsolved |

---

## Folder Structure (updated automatically by the end-of-day skill)

```
/00 Inbox                        ← unprocessed captures
/01 Daily Logs                   ← daily logs, named YYYY-MM-DD.md
    /C Logs                      ← Claude's own session logs, named [C] YYYY-MM-DD.md
/02 Projects/                    ← one folder per active project
/03 Projects Archive/            ← completed or paused projects
/04 Templates/                   ← note templates
README.md                        ← system overview (for humans)
CLAUDE.md                        ← this file (for Claude)
```

---

## Active Projects

### 1. FinOps
**Goal:** Consolidate JetX/Wash24h financial data into a single source of truth — P&L visibility by station/region/company, automated reconciliation, VAT compliance, and Xero integration.
**Why it exists:** Revenue lives in the JetX op system, costs are migrating to Xero, settlements from Vietinbank + Cybersource arrive in separate files, VAT matching against EasyInvoice is manual, and the finance team assembles P&L by hand. Needed for daily ops discipline and Series A-grade financial visibility.
**Key files:** `Project Overview (FinOps).md` for goal, outcomes, and open problems.
**Start a session:** Read `Project Overview (FinOps).md` to orient, then tackle the first open problem.

---

### 2. Jetx-Lakehouse
**Goal:** Wash24h/JetX data warehouse — medallion lakehouse (Bronze → Silver → Gold) consolidating 20+ PostgreSQL source tables into a single analytics-grade store; feeds FinOps, PowerBI, and CLI reporting.
**Why it exists:** 17+ stations, ~42K customers, ~5K washes/month — ad-hoc Pandas + raw PG queries are unauditable and fragile; medallion + dbt + Iceberg give tested, reproducible transforms and audit-grade numbers for Series A.
**Key files:** `Project Overview (Jetx-Lakehouse).md` (goal / why / open problems), `Current Architecture (Jetx-Lakehouse).md` (stack snapshot), `Roadmap (Jetx-Lakehouse).md` (sequenced plan), `Decision Log (Jetx-Lakehouse).md` (decisions from 2026-04-20 forward). `references/` is a frozen archive — do not edit.
**Start a session:** Read `Project Overview (Jetx-Lakehouse).md` to orient, then work from `Roadmap (Jetx-Lakehouse).md`.

---

### 3. Identity
**Goal:** Stand up and operate Keycloak as the single identity provider for every JetX app — one realm, capability-based RBAC, SSO, MFA. Shared spine service.
**Why it exists:** Platform principle #2 demands identity in one place. FinOps needs auth now, and every future app (Camera, CRM, Fleet) will depend on the same realm, capabilities, and onboarding runbook. Getting this right once makes every app cheaper and more secure.
**Key files:** `Project Overview (Identity).md` for goal, outcomes, and open problems. Implementation spec lives in `[[01-keycloak-setup]]` under the framework reference.
**Start a session:** Read `Project Overview (Identity).md` to orient, then tackle the first open problem (secret management strategy is the likely first).

---

### 4. Jetx-Tickets
**Goal:** *(Pending interview — fill in Project Overview.)*
**Why it exists:** *(Pending interview.)*
**Key files:** `Project Overview (Jetx-Tickets).md` for goal, outcomes, and open problems. Agent handoff uses `04 Templates/template-new-app-briefing.md`.
**Start a session:** Read `Project Overview (Jetx-Tickets).md` to orient. Note: this app is being built on the JetX framework — Keycloak + .NET 9 + SvelteKit + PG17. First blocking step is filing a Keycloak client proposal in the Identity project.

---

---

## Architecture Alignment (mandatory for every project)

Every active project in this vault is an instance of the JetX architecture. Projects do not invent their own rules — they declare how they align to the existing spine.

**The spine:**
- [[jetx-platform-principles]] — 10 platform rules (identity, data ownership, integration patterns)
- [[jetx-data-ownership-principles]] — three-channel model (sync API, command topic, change events)
- [[00-architecture]] — the stack (.NET 9 + SvelteKit + Keycloak + PG17 on self-hosted infra)

**Every `Project Overview` must contain a "See Also" or equivalent section linking at minimum to:**
1. `[[jetx-platform-principles]]` — and name which principles are most load-bearing for this project
2. `[[jetx-data-ownership-principles]]` — and declare this project's channel position (source? reader? writer via command? downstream sink?)
3. Any upstream / downstream active projects it depends on

**If a project needs to break a principle,** capture that in its `Decision Log` with the reasoning — don't silently diverge. A principle that's been challenged and held is stronger than one nobody tested.

**When scaffolding new projects** (via the `new-project` skill or by hand), add this alignment section before closing out the interview.

---

## What Claude Should Do in This Vault

1. **Read the project overview first** — every project has a `type: problems` note with the goal, why, outcomes, and open problems. Read it before doing anything else in that project.
2. **Read front matter** — it tells you note type, project, and relevance
3. **Look for patterns across notes** — surface connections Jimmy hasn't noticed
4. **Push toward output** — always end with "here's the next thing to make"
5. **Don't pad** — Jimmy wants concrete help, not summaries of what he already wrote
6. **Enforce architecture alignment** — when reviewing or scaffolding any project, verify its `Project Overview` has the See Also block linking to the spine. If missing, add it.

## Project Ownership & Agent Contribution Protocol

Some projects are driven directly by Jimmy — he owns the thinking, the decisions, and the canonical docs. Agents (Claude, Laika, Nemo, George, Milu, Kelly, or any future agent) must NOT silently edit canonical files in a Jimmy-driven project.

**Jimmy-driven projects (as of 2026-04-20):**

- [[Project Overview (Identity)]] — Keycloak realm design, capability schema, MFA/SSO policy, session tuning, and everything in the Identity project are Jimmy's to decide. Do not edit Project Overview, Principles Found, or the Decision Log without asking first.

**For every Jimmy-driven project, the canonical files are off-limits to direct edits:**

- `Project Overview (<Project>).md`
- `Principles Found (<Project>).md`
- `Decision Log (<Project>).md`
- Any file without an agent prefix in its filename or an `author:` field in its front matter

**How agents write back (input pattern):**

When an agent discovers something relevant — an observation, a question, a proposed decision, a risk, a finding — create a new note in the project folder instead of editing canonical docs. Use this shape:

- **Filename:** `[<AgentCode>] <Kind> — <Short Topic>.md`
  - Claude uses `[C]`; other agents use their own code (e.g. `[Nemo]`, `[Laika]`)
  - Kinds: `Question`, `Proposal`, `Finding`, `Risk`, `Note`
  - Examples: `[C] Proposal — MFA for staff only in v1.md`, `[Nemo] Finding — Keycloak admin console lacks audit log shipping.md`
- **Front matter:**
  ```
  ---
  type: agent-input
  project: Identity
  author: claude          # or laika, nemo, george, milu, kelly
  kind: question          # question | proposal | finding | risk | note
  status: open            # open | acknowledged | actioned | declined
  date: YYYY-MM-DD
  ---
  ```
- **Body:** lead with the ask — what Jimmy needs to decide or know. Then the evidence, then a proposed next step. Keep it one screen.

**How Jimmy processes them:** he reads, then either (a) promotes the content into `Project Overview` / `Decision Log` / `Principles Found`, (b) marks `status: acknowledged`, (c) marks `status: actioned` with a brief outcome note, or (d) `status: declined` with a reason. Agents should check their own prior inputs in the project folder before filing duplicates.

---

## What Claude Should NOT Do

- **Notes without the `[C]` prefix are user-created** — only edit them after asking the user first.
- **Do not edit canonical files in Jimmy-driven projects** (see Project Ownership section above) — file a `[<AgentCode>] <Kind> — <Topic>.md` input note instead.

## What Claude CAN Do

- **Create new notes anywhere in the vault**, including /02 Projects/ and /01 Daily Logs/
- **Claude's own session logs go in /01 Daily Logs/C Logs/** — named `[C] YYYY-MM-DD.md`

## Naming Convention for Claude-Created Notes

All notes created by Claude must follow this format:
- **Filename:** prefix with `[C] ` — e.g. `[C] Integration Setup.md`
- **Front matter:** include `author: claude` so it's queryable

This makes it immediately clear in the file explorer which notes are yours and which are Claude's output.

---

## Skills

These skills are installed in Cowork and trigger automatically based on what you say.

| Skill | Triggers | What It Does |
|---|---|---|
| `good-morning` | "good morning", "let's get to work", "start my day" | Reads vault, recaps recent work, recommends the most important thing, then helps pick what to work on (inbox or project) |
| `end-of-day` | "/eod", "wrap up", "end of day", "we're done for the day", "that's it for today" | Logs the session to `C Logs/`, updates the Folder Structure in this file so it stays accurate |
| `new-project` | "new project", "start a project", "create a project" | Interviews you, creates project folder + files, registers icons and pins, updates this file with the new project |

---

*This file is maintained by Jimmy. Update it when projects change.*
