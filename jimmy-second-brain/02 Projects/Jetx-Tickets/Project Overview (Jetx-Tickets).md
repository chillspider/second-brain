---
type: problems
date: 2026-04-20
project: Jetx-Tickets
author: claude
---
---

project: Tickets

status: in-development

created: 2026-04-20

last-updated: 2026-04-23

stack: keycloak, dotnet-9, sveltekit, python-lark

---

  

# Project Overview — Tickets

  

Internal ops ticketing for Wash24h. Staff create tickets via the web app or the Lark bot; supervisors claim, comment, resolve via interactive cards or the web UI.

  

**Replaces:** `jetx-lark-ticketing` (Python bot + Lark Base) — retired after repeated failures with the Lark v2 card form-container surface.

  

## Architecture Alignment

  

This project is the first JetX app to fully adopt the platform blueprint:

  

- **`jetx-platform-principles`** — Keycloak as sole auth IdP, JWT bearer on API, no per-app user table beyond role mapping. No tokens in client-side storage.

- **`jetx-data-ownership-principles`** — Platform `User` is upstream-only (FK UUID via Keycloak `sub`). Tickets app owns `Ticket`, `Comment`, `Category`, `UserRole`. No data duplication.

- **`00-architecture`** — Three services, one responsibility each: API (.NET), web (SvelteKit BFF), bot (Python Lark adapter). Postgres 17 shared host.

- **`01-keycloak-setup`** — Three clients per app: `tickets-web` (confidential, auth-code+PKCE), `tickets-api` (bearer-only), `tickets-bot` (service account). Group → capability mapping (`tickets.reporters|supervisors|admins` → `tickets:read|write`). Domain role kept separate in `user_roles` table for finer-grained authorization.

- **`02-dotnet-backend`** — Clean Architecture (Domain → Application → Infrastructure → Api). EF Core 9 + Npgsql. MediatR + FluentValidation + Result<T>. Testcontainers for integration tests.

- **`03-sveltekit-frontend`** — BFF pattern via `openid-client` + `iron-session`. Session cookie is encrypted, httpOnly, SameSite=Lax; tokens live server-side keyed by sid.

- **`04-deployment`** — **Deviation:** Docker Compose + `deploy-dev.sh` per-service (rsync + `docker compose up -d --build`) instead of the systemd+Caddy pattern in the platform brief. Same three services, same isolation, simpler iteration for the size of this app.

- **`05-conventions`** — Monorepo-per-service (`jetx-tickets-{api,web,bot}`), `push-dev.sh` / `deploy-dev.sh` script names, CLAUDE.md at repo root, Co-Authored-By trailer on every commit.

  

## Stack

  

| Layer | Tech |

|---|---|

| Identity | Keycloak 26.6.1 (Incus container, systemd) |

| API | .NET 9, Clean Architecture, EF Core 9, Postgres 17, MediatR 12, FluentValidation, Serilog, prometheus-net |

| Web | SvelteKit 2, Svelte 5 (runes), TypeScript strict, Tailwind 4, shadcn-svelte, @tanstack/svelte-query v6, zod, iron-session v8, openid-client v6 |

| Bot | Python 3.12, lark-oapi 1.5.3 (WS transport), httpx, pytest |

| Deploy | Docker Compose on dev server (`jimmy@100.100.133.64`) |

| Tests | xUnit + Testcontainers (.NET); Vitest + @testing-library/svelte + Playwright (web); pytest-asyncio (bot) |

  

## Data Model (owned tables)

  

```

tickets          id pk, ticket_number uniq, title, description, category_id fk,

                 priority enum, status enum, reporter_user_id, assignee_user_id,

                 created_at, claimed_at, escalated_at, resolved_at, deadline,

                 lark_channel_msg_id

comments         id pk, ticket_id fk (cascade), author_user_id, body, created_at

                 composite idx (ticket_id, created_at)

categories       id pk, name uniq, description, sla_hours, color, is_active

user_roles       user_id pk (= Keycloak sub), role enum, is_active, created_at,

                 name, email, preferred_username, updated_at

                 (identity fields cached from JWT on each authenticated request)

```

  

## Authorization

  

**Capabilities (Keycloak groups → API client roles):**

- `tickets:read`, `tickets:write` — assigned to `tickets.reporters`, `tickets.supervisors`, `tickets.admins`.

  

**Domain rules (Application-layer Policies.cs):**

  

| Action | Rule |

|---|---|

| Create ticket | user has any `user_roles` row |

| Claim | role ∈ {supervisor, admin} |

| Resolve | role ∈ {supervisor, admin} AND assignee == current |

| Reopen | reporter == current |

| Escalate / Reassign | role ∈ {supervisor, admin} |

| Comment | role ∈ {reporter, supervisor, admin} AND current is reporter or assignee |

| /admin/* | role == admin |

  

**Auto-provisioning:** first authenticated request from a Keycloak sub with no `user_roles` row creates one (role=`Reporter` by default; `Admin` if sub is in `Bootstrap__AdminSubs` env). Identity fields (name/email/preferredUsername) cached from JWT and re-synced on every request.

  

## Endpoints

  

```

GET  /health/live, /health/ready, /metrics

GET  /tickets?status&assignee&limit&offset

POST /tickets

GET  /tickets/{id}

POST /tickets/{id}/claim | /resolve | /reopen | /confirm-resolve | /escalate | /reassign

GET  /tickets/{id}/comments

POST /tickets/{id}/comments

GET  /user-roles (admin) | /user-roles/{userId} | /user-roles/me

POST /user-roles (admin)

PUT  /user-roles/{userId} (admin)

GET  /user-roles/directory (tickets:read — non-admin directory lookup)

GET  /categories | /categories/{id}

POST /categories (admin)

PUT  /categories/{id} (admin)

DELETE /categories/{id} (admin)

```

  

## Services (dev)

  

| Service | URL |

|---|---|

| Web | http://100.100.133.64:3050 |

| API | http://100.100.133.64:5050 |

| Keycloak | http://wash24h-identity-prod.taila02691.ts.net:8080 |

| Bot | no port — outbound Lark WS |

| Postgres | host localhost:5432, DB `tickets_app`, role `tickets_api` |

  

## Definition of Done (status as of 2026-04-23)

  

- [x] Sign in → Keycloak → `/tickets` (dev URL)

- [x] Full ticket lifecycle in web UI (create/list/claim/resolve/reopen/comment/escalate)

- [x] Lark bot — inbound commands (`/ticket`, `/my`) + card actions (claim/comment)

- [x] DM-style notifications on state change (channel-based via `LARK_SUPPORT_CHANNEL_ID`; personal DM deferred)

- [x] All three services deployed on `wash24h-hcm-a` (Docker, not systemd)

- [x] Backend tests green; GitHub Actions CI configured

- [~] E2E Playwright login → create → claim → resolve (test.fixme pending KC bypass fixture)

- [ ] Public HTTPS routing `tickets.app.wash24h.io` / `api.tickets.app.wash24h.io` — blocked on DNS + Traefik

- [x] Old `jetx-lark-ticketing` still functional as fallback (archive deferred until DNS flip)

  

## Notable decisions

  

- **Pivoted from Lark Base to Postgres** — Lark Base's rate limits and schema fragility made it unfit as primary store.

- **Pivoted from Lark cards to full web app** — Lark v2 cards' form-container has undocumented quirks that consumed days with no payoff.

- **Docker Compose instead of systemd** — simpler iteration for a three-service app on a single dev box; revisit for multi-node.

- **Reusable admin CRUD layer** — `createResourceQuery` / `createResourceMutations` + `<ResourceTable>` / `<ResourceDialog>` built in web repo with zero app-specific imports, ready to extract to `@jetx/svelte-admin` when a second app needs it.

- **Session split (web)** — iron-session cookie holds only `sid` + short PKCE flow; user identity + tokens in server-side Map. Required because Keycloak JWTs overflow browser's 4KB cookie cap.

  

## Repos

  

| Repo | Branch | Purpose |

|---|---|---|

| `jetx-tickets-api` | master | .NET 9 backend |

| `jetx-tickets-web` | main | SvelteKit BFF + UI |

| `jetx-tickets-bot` | master | Python Lark WS bot |

| `jetx-lark-ticketing` | master | **Archived** — predecessor reference only |