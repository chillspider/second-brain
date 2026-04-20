# JetX Enterprise Framework — Architecture Overview

> **Goal:** A self-hosted enterprise framework for JetX/Wash24h internal applications.
> **Stack:** .NET 9 backend (Clean Architecture) + SvelteKit frontend + Keycloak for identity.

---

## 1. Why this stack

**Not Superblocks / Retool / Appsmith.** Those platforms solve "non-engineers need to build apps without IT bottleneck." Our problem is the opposite: one engineer (+ small team) shipping many apps on custom infra we already control. A platform would fight our existing stack (Tailscale, Incus, MinIO, Grafana, PostgreSQL 17 on hcm-db-a).

**Not ABP Framework.** ABP is excellent but opinionated toward Blazor/Angular/MVC UIs. It would actively fight a SvelteKit frontend — we'd end up using ~40% of it and paying 100% of its learning-curve cost.

**Chosen path: Clean Architecture template + Keycloak + SvelteKit.**
- We own every line of code.
- Identity/RBAC/SSO/MFA is solved by Keycloak (one container, standard OIDC).
- Frontend is independent — SvelteKit talks to .NET API over HTTP, auth via OIDC.
- Fits existing infra patterns (systemd, Incus, Tailscale, Prometheus).

---

## 2. Component map

```
                 ┌─────────────────────────────────────────────────────┐
                 │                    Users                            │
                 │  (staff, admins, pi-ops via browser or mobile)      │
                 └───────────────────────────┬─────────────────────────┘
                                             │  HTTPS
                                             ▼
                 ┌─────────────────────────────────────────────────────┐
                 │  SvelteKit Frontend (app.wash24h.com)               │
                 │  - OIDC auth code flow + PKCE                       │
                 │  - Stores tokens in httpOnly cookies (server hooks) │
                 │  - Calls .NET API with Bearer access token          │
                 └─────────┬─────────────────────────────┬─────────────┘
                           │                             │
              OIDC redirect│                             │Bearer JWT
                           ▼                             ▼
         ┌──────────────────────────────┐   ┌──────────────────────────────┐
         │  Keycloak (auth.wash24h.com) │   │  .NET 9 API (api.wash24h.com)│
         │  - Single realm: jetx        │◄──┤  - Validates JWT via JWKS    │
         │  - Groups: pi-ops,admins,    │   │  - Clean Architecture layers │
         │    staff, operation-center   │   │  - EF Core → hcm-db-a        │
         │  - Clients: jetx-web,jetx-api│   │  - Policies per endpoint     │
         │  - PostgreSQL backing store  │   │                              │
         └──────────────┬───────────────┘   └──────────────┬───────────────┘
                        │                                  │
                        └──────────────┬───────────────────┘
                                       ▼
                      ┌────────────────────────────────┐
                      │  PostgreSQL 17 (hcm-db-a)      │
                      │  - keycloak DB                 │
                      │  - jetx_app DB                 │
                      │  (separate databases, same     │
                      │   server; monitored via        │
                      │   Prometheus postgres_exporter)│
                      └────────────────────────────────┘
```

---

## 3. Layer responsibilities

### Keycloak — does all identity work

- User store, password hashing, MFA, password reset emails
- OIDC token issuance (access + refresh + ID tokens)
- Group membership (maps to our existing `pi-ops`, `admins`, `staff`, `operation-center` teams)
- Role definitions (fine-grained permissions: `camera:read`, `site:write`, etc.)
- Audit log for all login/logout events
- Admin UI for user management (no custom code)

### .NET 9 API — does business logic

- Validates incoming JWTs (signature via Keycloak JWKS endpoint, issuer, audience, expiry)
- Maps Keycloak groups/roles onto ASP.NET Core `AuthorizationPolicy`
- Clean Architecture layers: `Domain` → `Application` → `Infrastructure` → `Api`
- EF Core to PostgreSQL for domain data
- No user/password tables — Keycloak owns that

### SvelteKit — does UI + BFF

- Public pages + authenticated app shell
- SvelteKit server hooks act as a Backend-for-Frontend: browser never touches tokens directly
- Refresh tokens handled server-side in httpOnly cookies
- Typed API client generated from OpenAPI spec emitted by .NET

---

## 4. Repository layout

```
jetx-enterprise/
├── docs/                              # This folder
│   ├── 00-architecture.md
│   ├── 01-keycloak-setup.md
│   ├── 02-dotnet-backend.md
│   ├── 03-sveltekit-frontend.md
│   ├── 04-deployment.md
│   └── 05-conventions.md
├── backend/
│   ├── src/
│   │   ├── JetX.Domain/              # Entities, value objects, domain events
│   │   ├── JetX.Application/         # Use cases, CQRS handlers, DTOs
│   │   ├── JetX.Infrastructure/      # EF Core, external services, Keycloak admin
│   │   └── JetX.Api/                 # ASP.NET Core host, controllers/minimal APIs
│   ├── tests/
│   └── JetX.sln
├── frontend/
│   ├── src/
│   │   ├── routes/
│   │   ├── lib/
│   │   │   ├── api/                  # Generated OpenAPI client
│   │   │   └── auth/                 # OIDC helpers
│   │   └── hooks.server.ts           # Token refresh, session guard
│   └── package.json
├── infra/
│   ├── keycloak/
│   │   ├── realm-export.json         # Source of truth for realm config
│   │   └── docker-compose.yml
│   └── migrations/                   # EF Core migrations output
└── README.md
```

---

## 5. Naming conventions (JetX-specific)

| Item                    | Convention                          | Example                              |
|-------------------------|-------------------------------------|--------------------------------------|
| Keycloak realm          | lowercase, single word              | `jetx`                               |
| Keycloak clients        | `<project>-<surface>`               | `jetx-web`, `jetx-api`, `jetx-mobile`|
| Keycloak groups         | match existing team names           | `pi-ops`, `admins`, `staff`          |
| .NET projects           | `JetX.<Layer>`                      | `JetX.Application`                   |
| Database names          | `<project>_<purpose>`               | `jetx_app`, `keycloak`               |
| Subdomains              | `<role>.wash24h.com`                | `api`, `app`, `auth`                 |
| Environment variables   | `JETX_<UPPER_SNAKE>`                | `JETX_KEYCLOAK_AUTHORITY`            |

---

## 6. Security model at a glance

- **Tokens:** Access token lifetime 5 min, refresh token 30 min (sliding, max 8h).
- **Secret handling:** No secrets in git. Use `dotnet user-secrets` for dev, `/etc/jetx/secrets.env` for prod (root:jetx 640).
- **Transport:** TLS everywhere including inside Tailscale (defense in depth). Keycloak, API, and frontend all behind Caddy or nginx with Let's Encrypt or internal CA.
- **CORS:** API accepts only known frontend origins. No wildcards.
- **CSRF:** Irrelevant because auth is via Bearer tokens on non-cookie endpoints. SvelteKit BFF endpoints use SameSite=Strict cookies.
- **Rate limiting:** `AspNetCoreRateLimit` or built-in `AddRateLimiter` on `/auth/*` and anonymous endpoints.

---

## 7. What to read next

1. `01-keycloak-setup.md` — stand up Keycloak, configure the realm, export config
2. `02-dotnet-backend.md` — scaffold the solution, wire auth, first endpoint
3. `03-sveltekit-frontend.md` — OIDC flow, protected routes, API client
4. `04-deployment.md` — Incus containers, systemd units, reverse proxy
5. `05-conventions.md` — code style, commits, migrations, testing
