---
type: problems
date: 2026-05-12
project: JetX Tech Vision
author: claude
driver: jimmy
---

> **Jimmy-driven project.** This Project Overview, Principles Found, and Decision Log are owned directly by Jimmy. Agents (Claude, Laika, Nemo, etc.) do **not** edit these files — file an input note instead: `[<AgentCode>] <Kind> — <Topic>.md` with front matter `type: agent-input`. See the "Project Ownership & Agent Contribution Protocol" section in [[CLAUDE]] for the full pattern.

## Goal
Clearly define the vision of what JetX Platform should be — and what it is.

## Why
This project is the holistic documentation that provides guard rails and guidelines for every sub-project within the JetX realm. Every future app, every agent-built feature, every architectural decision references this. Without it, sub-projects drift; with it, they compose.

## Tangible Outcomes
- Overview of vision and mission
- Detailed descriptions of the platform and tech stacks
- Architectural structure of the platform
- Principles governing all future development
- AI-first framework that lets AI agents build new software strictly aligned to the vision, architecture, and tech stacks
- Timeline / roadmap for the next 3 years

## Open Problems
1. **Get it out of Jimmy's head.** Right now the vision, architecture, principles, and roadmap live in Jimmy's head (the chief architect). The immediate work is to brain-dump each of the six Tangible Outcomes into written documents — rough first, refined later. Concretely, this means producing draft notes for:
   - Vision & mission
   - Platform & tech stack reference
   - Architectural structure
   - Governing principles
   - AI-first agent framework
   - 3-year roadmap

   Sharper problems (relationship to existing spine docs, scope of the AI framework, audience, versioning cadence, etc.) get filed as they surface — don't pre-invent them.

## Architecture Alignment

**This project does not align to the spine — it *defines* the spine.** Tech Vision is the source from which the existing spine docs ([[jetx-platform-principles]], [[jetx-data-ownership-principles]], [[00-architecture]]) will be folded in, extended, or superseded. The exact relationship (wrap / replace / sit-above) is itself one of the open problems to resolve here.

**Authoritative outputs (produced by this project):**
- Vision & mission doc
- Platform & stack reference
- Architectural structure
- Governing principles (likely absorbs or supersedes [[jetx-platform-principles]] and [[jetx-data-ownership-principles]])
- AI-agent framework spec
- 3-year roadmap

**Downstream consumers (every active project + every future project):**
- [[Project Overview (Identity)]]
- [[Project Overview (FinOps)]]
- [[Project Overview (Jetx-Lakehouse)]]
- [[Project Overview (Jetx-Tickets)]]

## See Also
- [[jetx-platform-principles]] — current 10 platform rules; this project will reconcile with or supersede them
- [[jetx-data-ownership-principles]] — current three-channel model; ditto
- [[00-architecture]] — current stack snapshot; will be folded into the platform/stack outcome
- [[Principles Found (JetX Tech Vision)]] — durable principles discovered while building this project
