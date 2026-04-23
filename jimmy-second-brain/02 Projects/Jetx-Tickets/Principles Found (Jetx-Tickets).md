---
type: note
project: Jetx-Tickets
author: claude
---
---

project: Tickets

type: principles-found

created: 2026-04-23

---

  

# Principles Found — Tickets

  

Patterns uncovered while building that may generalize to other JetX apps. Candidates for promotion to framework docs once a second app confirms them.

  

## Identity auto-provisioning on first authenticated request

  

A caller with a valid Keycloak JWT but no app-local role row should be auto-provisioned (with the lowest-privilege default) rather than rejected. Prevents a chicken-and-egg where admin UI needs a user row to exist before the user can log in to create their own row.

  

**Implementation:** middleware between `UseAuthentication` and `UseAuthorization` that resolves `sub` → `user_roles` row, creates with `Reporter` on miss, and re-syncs cached identity fields (name/email/preferredUsername) from JWT on every request. Sub → role resolution is O(1) (indexed pk lookup) so perf cost is negligible.

  

**Bootstrap admins** via `Bootstrap__AdminSubs` env CSV — app startup upserts those subs as `Admin`. Solves first-admin problem without hard-coding anything.

  

## Directory read endpoint, separate from user management

  

Non-admin viewers need to resolve `reporter_user_id` / `assignee_user_id` → display name. Don't gate the whole `/user-roles` admin list behind read-everyone — add a narrower `GET /user-roles/directory` (or `/users`) returning just `{ userId, name, preferredUsername }` under `tickets:read`. Email + role stay admin-only.

  

Framework should expose a `DirectoryService` abstraction so every app doesn't rebuild this.

  

## Session split for Keycloak-backed BFFs

  

Keycloak JWTs (access + refresh + id) push an encrypted session cookie past the 4 KB browser cap. Don't pack tokens into the cookie — keep tokens + user identity in a server-side keyed by a short `sid`; cookie holds only `sid` + short PKCE flow scratchpad.

  

**Dev-scale:** in-memory Map. **Prod:** Redis or similar. Swap is a one-file change in `src/lib/server/session.ts`.

  

This is likely the right pattern for any JetX app that uses SvelteKit BFF with Keycloak — worth a framework doc (`03-sveltekit-frontend` section: "Session storage").

  

## URL origin mismatch at the token-exchange / logout boundaries

  

When the service runs behind `network_mode: host` (or any proxy), SvelteKit's `event.url.origin` can resolve to `localhost` while the browser-facing URL (and therefore the registered OIDC redirect_uri / post_logout_redirect_uri) is the public host. `openid-client` uses `event.url.origin + event.url.pathname` as the `redirect_uri` sent during the token exchange, which then fails validation at Keycloak.

  

**Rule:** in the BFF, always normalize OIDC URLs against the known-good `KEYCLOAK_REDIRECT_URI` rather than trusting `event.url`. Applies to `authorizationCodeGrant`'s currentUrl and `post_logout_redirect_uri`.

  

## Cookie `Secure` flag derived from auth URL, not NODE_ENV

  

Standard advice says `Secure=true` when `NODE_ENV === 'production'`. Breaks in Tailscale-internal HTTP dev where NODE_ENV is production (Docker build) but the actual URL is http://. The browser silently drops the cookie → redirect loop that looks like a bug but is cookie policy.

  

**Rule:** derive `Secure` from whether the redirect URI is `https://`. Mirrors reality regardless of build env.

  

## Docker Compose deploy pattern is good-enough for 3-service apps

  

The platform brief specifies systemd + Caddy for .NET services. For a single-dev-box deployment of three services, Compose + host networking + per-service `deploy-dev.sh` is materially simpler (one `rsync + docker compose up -d --build`) with no meaningful cost. Recommendation: platform brief should acknowledge Docker Compose as an alternative for < 5 services on < 2 nodes.

  

## Domain role kept separate from Keycloak groups

  

Spec called for two coarse capabilities (`tickets:read`, `tickets:write`) on KC groups and a finer-grained domain role (`reporter`, `supervisor`, `admin`) in `user_roles`. This separation is the right move — KC group membership is platform-level access (is this person even allowed to touch the API); domain role is app-level authorization (can they claim a ticket).

  

Framework doc should make this two-layer model explicit: "don't try to encode domain-level permissions as KC groups."

  

## Reusable admin CRUD pair (<ResourceTable> + hooks)

  

Built a generic `createResourceQuery<T>` / `createResourceMutations<T>` over `@tanstack/svelte-query` + `<ResourceTable>` / `<ResourceDialog>` / `<ResourceField>` components that lets a new admin page ship in ~50 LOC of glue. Ready for extraction to `@jetx/svelte-admin` npm package when a second app needs it.

  

The key architectural discipline: **zero app-specific imports inside `src/lib/api/` and `src/lib/components/admin/`** — toast surface is injected, resource names are parameters. Enforce this in framework docs for any "reusable" folder.

  

---

  

**Next step when Tickets stabilizes:** review this list, promote the validated items to the framework docs (`01-keycloak-setup`, `03-sveltekit-frontend`, `04-deployment`), delete the rest.