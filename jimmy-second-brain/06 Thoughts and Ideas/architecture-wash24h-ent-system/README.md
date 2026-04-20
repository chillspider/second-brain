# JetX Enterprise Framework — Docs

Internal guidance for building JetX/Wash24h applications on a self-hosted enterprise framework.

**Stack:** .NET 9 · SvelteKit · Keycloak · PostgreSQL 17

## Read in order

1. [00-architecture.md](./00-architecture.md) — Why this stack, component map, repo layout
2. [01-keycloak-setup.md](./01-keycloak-setup.md) — Identity provider: realm, groups, roles, clients
3. [02-dotnet-backend.md](./02-dotnet-backend.md) — Clean Architecture solution, JWT validation, CQRS
4. [03-sveltekit-frontend.md](./03-sveltekit-frontend.md) — BFF pattern, OIDC flow, typed API client
5. [04-deployment.md](./04-deployment.md) — Incus, systemd, Caddy, monitoring, backups
6. [05-conventions.md](./05-conventions.md) — Code style, git, migrations, secrets, dos and don'ts

## Quick reference

| Need                                 | Go to                                                  |
|--------------------------------------|--------------------------------------------------------|
| Add a new permission                 | `01-keycloak-setup.md` §3 + `02-dotnet-backend.md` §4  |
| Wire a new endpoint                  | `02-dotnet-backend.md` §5                              |
| Protect a new route                  | `03-sveltekit-frontend.md` §7 + §11                    |
| Ship to production                   | `04-deployment.md` §6                                  |
| Add a database table                 | `02-dotnet-backend.md` §8 + `05-conventions.md` §3     |
| Rotate a secret                      | `04-deployment.md` §9                                  |

## Principles

- [[jetx-platform-principles]] — 10 rules every app obeys
- [[jetx-data-ownership-principles]] — three-channel model for data flow

## Applied In

Projects building this framework:

- [[Project Overview (Identity)]] — implements [[01-keycloak-setup]]; the spine service every app depends on
- [[Project Overview (FinOps)]] — first end-to-end consumer of the framework (Keycloak + JWT + capability scopes + BFF)
- [[Project Overview (Jetx-Lakehouse)]] — data warehouse consuming from every source system

## Status

These docs are drafts. Update them as the code evolves — if a doc lies, the doc is the bug.
