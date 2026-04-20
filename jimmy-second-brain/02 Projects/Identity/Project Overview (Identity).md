---
type: problems
date: 2026-04-20
project: Identity
author: claude
driver: jimmy
---

> **Jimmy-driven project.** This Project Overview, Principles Found, and Decision Log are owned directly by Jimmy. Agents (Claude, Laika, Nemo, etc.) do **not** edit these files — file an input note instead: `[<AgentCode>] <Kind> — <Topic>.md` with front matter `type: agent-input`. See the "Project Ownership & Agent Contribution Protocol" section in [[CLAUDE]] for the full pattern.

## Goal
Stand up and operate Keycloak as the single identity provider for every JetX app — one realm, capability-based RBAC, SSO, and MFA. Everything auth-adjacent lives here; no app re-implements identity.

## Why
Platform principle #2 is absolute: "Identity is solved in exactly one place." FinOps needs real auth now (finance-team capability scopes); Jetx-Lakehouse, Camera, CRM, and Fleet will follow. Doing this once — realm design, client registration, capability schema, MFA rollout — prevents each app from rolling its own and makes SSO/audit/offboarding trivially uniform. Spec already exists in [[01-keycloak-setup]]; this project turns it into running infrastructure and an ongoing operations practice.

## Tangible Outcomes
- Keycloak 26.x running in Incus on `wash24h-hcm-a`, backed by PostgreSQL 17 on `hcm-db-a` (separate `keycloak` database)
- Reverse-proxied at `auth.wash24h.com` via Caddy; Prometheus scraped
- Realm `jetx` configured: groups mirror teams, realm roles for coarse access, client roles for fine-grained capabilities
- Clients registered: `jetx-web` (SvelteKit BFF, confidential), `jetx-api` (.NET backend, bearer-only), plus one client per app
- Capability schema documented: `<resource>:<action>` (e.g. `sites:read`, `invoices:write`, `finance:close`)
- MFA policy defined and enforced for staff/admin
- Realm export committed to repo; import procedure proven on a fresh environment
- App-onboarding runbook — "how to register a new app as a Keycloak client in under 30 minutes"
- Operational runbook — rotation, backups, incident response if Keycloak is down
- FinOps onboarded as the first real consumer

## Open Problems
1. Secret management strategy — where do Keycloak admin password, DB password, client secrets live (SOPS? Vault? systemd-creds?)
2. MFA scope — all users, or staff/admin only to start; which factor (TOTP, WebAuthn, both)
3. SSO — federate Google Workspace? Zalo? Keep everything local for v1?
4. Session + token lifetime tuning — access token (short), refresh token (sliding), SSO session (long); exact values TBD
5. Capability schema governance — who owns the registry of `resource:action` pairs as apps multiply
6. Group → role mapping discipline — keep groups as team-shaped bundles, never as ad-hoc role proxies
7. Backup strategy — realm export cadence, DB backup frequency, restore drill
8. "Keycloak down" contingency — does any app need read-only degraded mode? Token cache grace period?
9. User provisioning pipeline — manual in admin console for v1, or SCIM / HR system integration later
10. Audit log retention and shipping to central logging

## Architecture Alignment

**Platform principles ([[jetx-platform-principles]]):**
- **#1 The platform is the spine** — Keycloak IS part of the spine; it must be boring, stable, and not an app
- **#2 Identity in one place** — this project is the literal implementation of #2
- **#3 Permissions are capabilities** — the capability schema defined here is consumed by every app's policy layer
- **#7 Boring tech wins** — Keycloak 26.x, standard OIDC, PG17; no custom auth code anywhere

**Data-ownership channel position ([[jetx-data-ownership-principles]]):**
- **Channel 1 (Sync API)** — admins edit users/groups/roles via the Keycloak admin console (human-in-the-loop)
- **Channel 3 publisher** — token content (sub, groups, client roles) is effectively a read-only API every app depends on
- **Never Channel 2** — Keycloak is not driven by command topics; no other system writes to it asynchronously

**Upstream / downstream dependencies:**
- **Downstream consumers (every app eventually):** [[Project Overview (FinOps)]] (first real client), future Jetx-Lakehouse admin UI, Camera, CRM, Fleet
- **Infra dependency:** shared PostgreSQL 17 (`hcm-db-a`), Caddy, Prometheus on `wash24h-hcm-a`

## Source Docs
- [[01-keycloak-setup]] — the canonical setup spec (framework reference; treat as implementation spec for this project)

## See Also
- [[Project Overview (FinOps)]] — first consumer; its capability schema needs a home here
- [[jetx-platform-principles]] · [[jetx-data-ownership-principles]] · [[00-architecture]]
- [[Principles Found (Identity)]]
