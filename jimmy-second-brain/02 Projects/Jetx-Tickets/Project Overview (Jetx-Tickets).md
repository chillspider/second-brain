---
type: problems
date: 2026-04-20
project: Jetx-Tickets
author: claude
---

## Goal
*(Pending — fill in once Jimmy answers the interview.)*

## Why
*(Pending — fill in once Jimmy answers the interview.)*

## Tangible Outcomes
*(Pending — fill in once Jimmy answers the interview.)*

## Open Problems
1. (to be defined)

## Architecture Alignment

This project is being built on the JetX framework — Keycloak identity + .NET 9 backend + SvelteKit frontend + PG17, deployed on `wash24h-hcm-a` via Incus + systemd + Caddy. Engineering handoff uses the [[template-new-app-briefing]] template.

**Platform principles ([[jetx-platform-principles]]):**
- #1 Apps stay independent — Jetx-Tickets is its own repo, its own database, its own release cycle
- #2 Identity in one place — auth via [[Project Overview (Identity)|Identity (Keycloak)]]; no login UI, no password table
- #3 Permissions as capabilities — capability scopes TBD during scoping
- #4 Core domain in one service — reference `User`, `Site`, `Customer`, `Vehicle` by ID via platform-core
- #5 Each app owns its own domain data — Jetx-Tickets owns tickets, SLAs, and any ticket-adjacent schemas
- #7 Boring tech — standard stack, no new runtimes, no experimental frameworks

**Data-ownership channel position ([[jetx-data-ownership-principles]]):**
- *(To be declared — what does Jetx-Tickets read sync vs. via events? What does it publish? Fill in after interview.)*

**Upstream / downstream dependencies:**
- **Identity provider:** [[Project Overview (Identity)]] — must register `jetx-tickets-web` and `jetx-tickets-api` clients here before any code
- **Upstream data:** *(TBD — probably platform-core for User/Site/Customer, possibly Jetx-Lakehouse for reporting)*
- **Downstream consumers:** *(TBD)*

## Engineering Handoff

- **Briefing template:** [[template-new-app-briefing]] — fill the `{{PLACEHOLDER}}` fields and send to the agent before they start
- **First blocking step:** file a Keycloak client proposal in `02 Projects/Identity/` as per the briefing
- **Definition of done:** see Section 8 of the briefing template

## See Also
- [[template-new-app-briefing]] — agent briefing for this build
- [[Project Overview (Identity)]] — Keycloak provider (blocking dependency)
- [[jetx-platform-principles]] · [[jetx-data-ownership-principles]] · [[00-architecture]]
- [[Principles Found (Jetx-Tickets)]]
