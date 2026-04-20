# JetX Enterprise Platform — Design Principles

> One page. The rules that govern every app, every service, every decision.

---

## North Star

**One portal, many apps, shared spine.** Users see one JetX. Engineers keep apps independent. The platform owns identity, core domain data, and the surrounding fabric — nothing more.

---

## The 10 Principles

**1. Apps stay independent; the platform is the spine.**
FinOps, Camera, CRM, Fleet each own their repo, database, release cycle. The platform is Keycloak + core domain service + portal shell + event bus. Never merge apps into one codebase to solve an integration problem — fix the spine instead.

**2. Identity is solved in exactly one place.**
Keycloak. Every app validates the same JWT. No app stores passwords, no app implements login, no app has its own user table keyed by email. Apps reference users by Keycloak `sub` (UUID).

**3. Permissions are capabilities, not roles.**
Client roles in Keycloak express `resource:action` (`sites:read`, `invoices:write`). Apps enforce policies on capabilities. Roles (`ops`, `admin`, `staff`) are just bundles of capabilities assigned via groups. Never check `user.role == "admin"` in code.

**4. Core domain lives in one service.**
`platform-core` owns `User`, `Site`, `Customer`, `Vehicle`, `Organization` — entities multiple apps need. Apps FK into core by ID and call its API for canonical data. Apps never duplicate these tables. If you're tempted to, add a field to core instead.

**5. Each app owns its own domain data.**
FinOps owns invoices, Camera owns streams and events, CRM owns deals and pipelines. No cross-app database reads. No shared schemas beyond core. If App A needs App B's data, App A calls App B's API.

**6. Integration pattern: start with reverse proxy + iframe, graduate only when it hurts.**
Portal mounts each app at a path. JWTs shared via domain cookie. Zero refactoring on day one. Only move to path-mounted shared layout when iframe UX pain is real. Only move to Module Federation when you have multiple teams fighting over one repo — not before.

**7. Cross-app communication is asynchronous by default.**
Apps emit domain events (`invoice.paid`, `plate.detected`, `site.activated`) to NATS/RabbitMQ. Other apps subscribe to what they care about. No direct app-to-app API calls for workflows — only for authoritative reads. This keeps apps independently deployable.

**8. Deep links are URL conventions, not a framework.**
`/crm/customers/{id}` → a button → `/camera/events?customer={id}`. Every entity has a canonical URL under its owning app. Other apps link to it. No "linking engine," no registry service. Just stable URLs and discipline.

**9. One design system, copied not imported.**
shadcn-svelte components live in `shared/design-system` as source, consumed by each app. Apps can diverge when they must, but diverge deliberately. The portal enforces a consistent shell (top bar, user menu, notifications) around whatever the app renders inside.

**10. The platform is boring infrastructure, not a product.**
No in-house framework abstractions. No "JetX SDK" wrapping .NET. No custom ORM. No bespoke auth. Every piece is something a new hire can google. Innovation budget is spent on apps that serve the business, not on platform cleverness.

---

## What the platform provides

| Capability       | Owner                      | Consumed how                       |
|------------------|----------------------------|------------------------------------|
| Identity, SSO    | Keycloak                   | OIDC → JWT in every app            |
| Permissions      | Keycloak + per-app policy  | JWT claims → ASP.NET Core policies |
| Core entities    | `platform-core` (.NET API) | REST + OpenAPI client per app      |
| Events           | NATS or RabbitMQ           | Publish/subscribe per bounded ctx  |
| Portal shell     | SvelteKit `portal` app     | Iframe / path mount                |
| Design system    | `shared/design-system`     | Copied into each app               |
| Audit log        | `platform-core`            | Append-only via API                |
| File storage     | MinIO                      | Presigned URLs from each app       |
| Search (later)   | Meilisearch or OpenSearch  | Per-app indexing                   |
| Monitoring       | Prometheus + Grafana       | Scrape `/metrics` on every service |

---

## What the platform does NOT provide

- A universal data model for business logic. Each app defines its own.
- A no-code / low-code builder. Engineers ship code.
- A workflow engine. Apps own their own workflows; cross-app coordination is via events.
- A unified frontend framework beyond the shell. Apps can use SvelteKit, Next, Blazor — portal doesn't care as long as they honor the shell contract.
- Backend abstraction layers. Each app talks to its own database directly.

---

## Decision tree for new capabilities

```
New feature request
   │
   ├─ Does one app own this domain? ───── YES ──► Build in that app. Done.
   │         NO
   │
   ├─ Do 2+ apps need the same entity? ── YES ──► Add to platform-core.
   │         NO
   │
   ├─ Is it cross-app workflow? ───────── YES ──► Domain event + subscribers.
   │         NO
   │
   └─ Is it portal chrome? (nav, notif) ─ YES ──► Build in portal shell.
                                           NO  ──► You probably don't need it yet.
```

---

## Phased rollout

1. **Foundation** — Keycloak, `platform-core`, one app (FinOps) migrated end-to-end. Prove the pattern.
2. **Portal shell** — SvelteKit portal + app launcher. FinOps runs inside it.
3. **Migration** — Move Camera, CRM, Fleet, Operations under portal one at a time. Each migration is: swap to Keycloak JWT → FK to `platform-core` → register in portal launcher.
4. **Events** — Add NATS/RabbitMQ when the first real cross-app workflow lands (not before).
5. **Polish** — Global search, unified notifications, audit UI.

Do not skip stages. Do not build stage 4 before stage 2 lands in production.

---

## Red flags that mean you're off the path

- A feature requires changing schemas in multiple app DBs simultaneously → you missed an event or a core entity.
- An app needs Keycloak admin API to do its work → permission model is wrong; fix the claims.
- The portal contains business logic → move it to an app or to `platform-core`.
- Two apps have the same table with slightly different columns → promote to `platform-core` now.
- You're writing a "platform framework" in C# or TypeScript → stop; pick a boring existing tool.
- A release of the platform blocks all app releases → coupling leaked; audit what changed.

---

## One-line summary

**Independent apps, one login, one core, one shell, boring tech, ship.**

---

## Applied In

Every active project aligns to these principles. If a project conflicts with a principle, either the project changes or the principle gets challenged explicitly in its Decision Log.

- [[Project Overview (Identity)]] — the literal implementation of #2 (identity in one place); owns the capability schema for #3
- [[Project Overview (FinOps)]] — owns invoices (#5), validates JWT from Keycloak (#2), uses capability scopes (#3), treats Xero as Channel 3 downstream; first app to implement the full stack
- [[Project Overview (Jetx-Lakehouse)]] — reads change events from every source (Channel 3 reader), materializes core domain data (#4) as dim_stations / dim_customers / dim_vehicles, never writes back

## See Also

- [[jetx-data-ownership-principles]] — the three-channel model these principles depend on
- [[00-architecture]] — the stack that implements these principles
- [[README]] — framework docs in reading order
