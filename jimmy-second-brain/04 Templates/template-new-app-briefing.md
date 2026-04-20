---
type: template
author: claude
purpose: Agent briefing to hand off when starting a new app on the JetX framework
---

# Briefing: Build {{APP_NAME}} on the JetX Framework

> Paste this to the engineering agent as their starting prompt. Fill every `{{PLACEHOLDER}}` before sending. Do not remove the non-negotiable sections — the agent will be held to them.

---

## 0. Your Role

You are the engineering agent building **{{APP_NAME}}**, a new JetX app on the shared platform. You are NOT starting from scratch — the platform (Keycloak identity, core domain service, portal shell, deployment fabric) already exists. Your job is to implement the app itself, obey the framework, and write back clearly when you need a decision.

---

## 1. The App

| Field | Value |
|---|---|
| **Name** | {{APP_NAME}} |
| **One-liner** | {{ONE_LINE_PURPOSE}} |
| **Primary users** | {{USERS — e.g. finance team, ops, customers}} |
| **App repo** | {{REPO_URL or TBD}} |
| **Target domain** | `{{SUBDOMAIN}}.wash24h.com` |
| **Owns this data** | {{OWNED_ENTITIES}} |
| **Reads from** | {{UPSTREAM_SERVICES_OR_DATA}} |
| **Emits / writes to** | {{DOWNSTREAM_CONSUMERS_OR_SYSTEMS}} |

---

## 2. Non-Negotiable — Read These First

Stop and read before writing any code:

1. [[jetx-platform-principles]] — 10 rules every app obeys
2. [[jetx-data-ownership-principles]] — three-channel model (sync API, command topic, change events)
3. [[00-architecture]] — component map and why this stack
4. [[05-conventions]] — code style, commit discipline, secrets

Confirm to Jimmy in your first message that you've read all four. If any principle looks wrong for your app, log it as a Decision Log entry in `02 Projects/{{APP_NAME}}/` and ask — do NOT silently diverge.

---

## 3. The Stack (no deviations without a Decision Log entry)

- **Backend:** .NET 9, Clean Architecture template, ASP.NET Core minimal APIs, EF Core → PostgreSQL 17
- **Auth:** Keycloak JWT validation; NEVER your own login, password table, or session scheme
- **Frontend:** SvelteKit (Node adapter), **BFF pattern** — OIDC Authorization Code + PKCE, tokens live server-side only in an encrypted httpOnly session cookie
- **Database:** PostgreSQL 17 on `hcm-db-a`, one dedicated database per app
- **Deployment:** systemd units on `wash24h-hcm-a`, reverse-proxied via Caddy with auto-HTTPS
- **Monitoring:** Prometheus scrape endpoints on both backend (`/metrics`) and frontend
- **Secrets:** systemd-creds / SOPS — never in the repo, never in plain env files checked in

---

## 4. Reference Docs (read in this order)

| # | Doc | Why |
|---|---|---|
| 1 | [[00-architecture]] | Component map, repo layout |
| 2 | [[01-keycloak-setup]] | Realm, groups, clients — understand the auth model |
| 3 | [[02-dotnet-backend]] | Clean Architecture, JWT validation, CQRS patterns |
| 4 | [[03-sveltekit-frontend]] | BFF, OIDC flow, typed API client |
| 5 | [[04-deployment]] | systemd units, Caddy routing, Prometheus |
| 6 | [[05-conventions]] | Code style, migrations, git |

---

## 5. Sequenced Tasks (do not reorder)

### Step 1 — Request Keycloak clients (BLOCKING)

You cannot self-register Keycloak clients. [[Project Overview (Identity)]] is a Jimmy-driven project. Before writing any code, file a Proposal note:

**File:** `02 Projects/Identity/[<YourAgentCode>] Proposal — {{APP_NAME}} Keycloak clients.md`

```markdown
---
type: agent-input
project: Identity
author: <your-agent-code>
kind: proposal
status: open
date: YYYY-MM-DD
---

## Ask
Register two Keycloak clients for {{APP_NAME}} in realm `jetx`.

### Client 1: {{app-slug}}-web (SvelteKit BFF)
- Access type: confidential
- Standard flow + service accounts: enabled
- Direct access grants: disabled
- Valid redirect URIs: https://{{SUBDOMAIN}}.wash24h.com/auth/callback
- Web origins: https://{{SUBDOMAIN}}.wash24h.com

### Client 2: {{app-slug}}-api (.NET backend)
- Access type: bearer-only
- No redirect URIs

### Capability scopes (client roles on {{app-slug}}-api)
- `{{resource1}}:read`
- `{{resource1}}:write`
- `{{resource2}}:read`
- `{{resource2}}:write`

### Group → role mapping
- Group `{{team-name}}` gets `{{capabilities}}`

## Why
{{2–3 lines: authorization model for this app}}

## Impact if delayed
Blocks login, blocks API calls from frontend, blocks deployment.
```

Wait for Jimmy to mark `status: actioned` before proceeding.

### Step 2 — Scaffold the .NET backend

Follow [[02-dotnet-backend]] section by section. Non-negotiables:

- **Solution layout:** `JetX.{{App}}.Domain`, `JetX.{{App}}.Application`, `JetX.{{App}}.Infrastructure`, `JetX.{{App}}.Api`, plus xUnit test projects
- **JWT validation:** use the JWKS endpoint from Keycloak (`https://auth.wash24h.com/realms/jetx/protocol/openid-connect/certs`); validate `iss`, `aud`, `exp`, signature
- **Authorization:** **capabilities, never roles.** `[Authorize(Policy = "invoices:write")]`, not `[Authorize(Roles = "admin")]`. Policies read `resource_access.{{app-slug}}-api.roles` from the JWT
- **EF Core + PostgreSQL:** `Guid.CreateVersion7()` for all new PKs; forward-only migrations
- **Logging:** Serilog with trace / span IDs; structured JSON output
- **Health:** `/health/live` (process is up) and `/health/ready` (dependencies reachable)
- **Metrics:** `/metrics` Prometheus endpoint (OpenTelemetry or prometheus-net)
- **Core domain FKs:** reference `User`, `Site`, `Customer`, `Vehicle`, `Organization` by UUID only; never duplicate their tables

### Step 3 — Scaffold the SvelteKit frontend

Follow [[03-sveltekit-frontend]]. Non-negotiables:

- **Adapter:** `@sveltejs/adapter-node` (not Vercel, not static)
- **BFF:** tokens live in an encrypted httpOnly session cookie — browser never sees access or refresh tokens
- **OIDC library:** `openid-client` on the server
- **API client:** typed, server-side, attaches `Authorization: Bearer <access>` from the session
- **Runes:** Svelte 5 runes syntax (`$state`, `$derived`, `$effect`)
- **TypeScript:** strict mode on; no `any`, use `unknown` and narrow

### Step 4 — Wire auth end-to-end and verify

Prove the whole flow works:

1. Unauthenticated browser hits `https://{{SUBDOMAIN}}.wash24h.com/` → redirected to Keycloak login
2. Returns with auth code → SvelteKit server exchanges for tokens → session cookie set
3. Browser makes API call → SvelteKit proxies to `api.{{SUBDOMAIN}}.wash24h.com` with bearer token
4. Backend validates JWT → enforces capability policy → returns data

**Verification checklist:**
- [ ] Protected endpoint returns 401 without token
- [ ] Protected endpoint returns 200 with valid token + right capability
- [ ] Protected endpoint returns 403 with valid token but missing capability
- [ ] Token refresh happens silently when access token nears expiry
- [ ] Logout clears session cookie AND hits Keycloak `end_session_endpoint`

### Step 5 — Deploy to the dev server

Follow [[04-deployment]]:

- systemd units: `{{app-slug}}-api.service`, `{{app-slug}}-web.service`
- Caddy routes: `api.{{SUBDOMAIN}}.wash24h.com` → backend, `{{SUBDOMAIN}}.wash24h.com` → frontend
- Secrets via systemd-creds or SOPS; never in plain env files
- Prometheus scrape config entries for both services
- Database: create `{{app-slug}}_app` database on `hcm-db-a`, separate user with least privilege
- Migration runbook in the repo's `docs/runbook.md`

### Step 6 — Tests

- **Backend:**
  - xUnit unit tests on Domain and Application
  - xUnit integration tests on Api (using `WebApplicationFactory<>` + Testcontainers for PG)
  - Auth tests: 401 / 200 / 403 cases with real JWT fixtures
- **Frontend:**
  - Vitest component tests
  - Playwright e2e for the auth flow — "user can log in, see home, log out"
- **CI:** tests must pass before any merge to `main`

### Step 7 — Register the project in Jimmy's vault

Before you consider yourself done, create the project folder in the vault:

- `02 Projects/{{APP_NAME}}/Project Overview ({{APP_NAME}}).md` — `type: problems`, with goal / why / tangible outcomes / open problems
- **Architecture Alignment block** — required per [[CLAUDE]]. Name the load-bearing platform principles, declare channel position per [[jetx-data-ownership-principles]], list upstream / downstream dependencies
- `02 Projects/{{APP_NAME}}/Principles Found ({{APP_NAME}}).md` — stub
- Update [[CLAUDE]] Active Projects with the new entry
- Update the framework `README.md` "Applied In" list

### Step 8 — App-level documentation

In the app's repo (not Jimmy's vault), create:

- `README.md` — 2-paragraph intro, link to [[00-architecture]], quickstart (clone / run / test)
- `docs/auth.md` — capability scopes this app uses, how to request a new one from Identity
- `docs/runbook.md` — deploy, rollback, health check, common failure modes
- `docs/architecture.md` — app-specific design decisions (keep short; link to framework docs instead of duplicating)

Do NOT duplicate framework content. Link to it.

---

## 6. Alignment Checklist (self-review before any PR)

Run through this before requesting review:

- [ ] Identity is via Keycloak only — no login UI, no password table, no hand-rolled JWT
- [ ] Authorization checks capabilities (`sites:read`, `invoices:write`), not roles (`user.role == "admin"`)
- [ ] Core domain entities (`User`, `Site`, `Customer`, `Vehicle`, `Organization`) are referenced by ID only; no duplicated tables
- [ ] Channel position declared in Project Overview — what do you read sync vs. via events? What do you write via commands vs. sync?
- [ ] `/health/live`, `/health/ready`, `/metrics` endpoints present on the backend
- [ ] No secrets in the repo; systemd-creds / SOPS used
- [ ] `Guid.CreateVersion7()` for all new PKs
- [ ] Migrations forward-only; reversals via compensating migrations
- [ ] PRs reference the principle or convention they're honoring on judgment calls

---

## 7. How to Write Back to Jimmy

Follow the Agent Contribution Protocol in [[CLAUDE]].

- **Questions / proposals / findings / risks** → file to the relevant project folder as `[<YourAgentCode>] <Kind> — <Topic>.md` with front matter `type: agent-input`, `status: open`
- **Auth / Keycloak matters** → `02 Projects/Identity/`
- **Framework-level observations** (a principle needs sharpening, a convention is ambiguous) → `06 Thoughts and Ideas/architecture-wash24h-ent-system/` as a `[<YourAgent>] Finding — ...md`
- **App-specific progress** → notes in `02 Projects/{{APP_NAME}}/`

Do not edit Jimmy-driven canonical files (Project Overview / Principles Found / Decision Log) directly — always file an input note.

---

## 8. Definition of Done (v1)

- [ ] User lands on `https://{{SUBDOMAIN}}.wash24h.com/`, sees Keycloak login, signs in, lands on app home
- [ ] Backend validates JWT and enforces capability policies on every protected endpoint
- [ ] Core domain entities referenced by ID only — no duplicated core tables
- [ ] All three data-ownership channels declared and implemented per the principles
- [ ] Deployed to dev server; systemd units stable; Caddy routing working; Prometheus scraping OK
- [ ] Tests passing in CI
- [ ] `README.md` + `docs/auth.md` + `docs/runbook.md` live in the app repo
- [ ] `02 Projects/{{APP_NAME}}/` exists in Jimmy's vault with Project Overview + Architecture Alignment
- [ ] [[CLAUDE]] Active Projects and framework `README.md` "Applied In" both list `{{APP_NAME}}`
- [ ] Alignment Checklist (Section 6) fully ticked
- [ ] Jimmy has reviewed and signed off

---

## 9. What You DO NOT Do

- Invent your own auth — ever
- Check roles in code (`user.role == "admin"`)
- Store tokens in browser `localStorage` or `sessionStorage`
- Spin up a new database for `User` / `Site` / `Customer` / `Vehicle` / `Organization` — FK into platform-core
- Add a "just for now" auth bypass
- Mix commands (async writes) and sync APIs on the same endpoint
- Duplicate framework docs in your app's repo
- Merge to `main` without the Alignment Checklist passing
- Silently break a principle — always log and ask

---

*When ready, reply with: "I've read all four spine docs ([[jetx-platform-principles]], [[jetx-data-ownership-principles]], [[00-architecture]], [[05-conventions]]) and I'm starting with Step 1 — filing the Keycloak client request. My agent code is {{AgentCode}}."*
