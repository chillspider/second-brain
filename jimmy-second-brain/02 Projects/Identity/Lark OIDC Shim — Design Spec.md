---

title: Lark OIDC Shim — Design Spec

status: approved-for-planning

date: 2026-04-20

owner: jimmy

author: claude-code (brainstorm partner)

supersedes: none

---

  

# Lark OIDC Shim — Design

  

A small, stateless Go service that adapts Lark's non-OIDC OAuth2 + Contacts API into a standards-compliant OIDC surface, so Keycloak can federate Lark as a normal OIDC IdP.

  

## 1. Goals & Non-Goals

  

### Goals (v1)

  

- Let JetX users click "Login with Lark" on Keycloak and be authenticated end-to-end.

- Provision users just-in-time in Keycloak on first Lark login (sync mode `import`).

- Keep Lark knowledge **confined to the shim**. Keycloak, apps, and the platform spine have no Lark-specific code.

- Remain operationally boring: single Go binary, k3s Deployment, standard OIDC surface, standard Traefik ingress with ACME TLS.

  

### Explicitly Non-Goals (deferred to v2+)

  

- Automatic department sync from Lark → Keycloak groups.

- Automatic offboarding when a Lark account is frozen/resigned.

- Role or capability assignment. v1 assigns groups and client roles manually in Keycloak after JIT provisioning.

- Multiple upstream IdPs behind one shim. v1 is one shim per IdP.

- Signing-key rotation automation. v1 rotation is manual (Secret update + `kubectl rollout restart`).

- Rate limiting, replay protection, or audit logging beyond what Keycloak and OAuth2 already provide.

  

## 2. Rejected Alternatives

  

- **Custom Keycloak Java SPI (Service Provider Interface).** Violates platform principle #7 (boring tech wins). Introduces JVM build pipeline and upgrade coupling between Keycloak and Lark-specific code inside the IdP.

- **Twisting Keycloak's generic OAuth2 broker into Lark shape.** Keycloak's generic broker assumes OIDC conventions Lark doesn't follow (no discovery doc, no `id_token`, different userinfo shape). Making this work requires custom mappers and is fragile against Lark API changes.

  

## 3. Architecture

  

### Components

  

- **Keycloak** (existing) — stays in its Incus container at `identity.app.wash24h.io`. Configured with a new "OpenID Connect v1.0" identity provider named `lark`. Untouched otherwise.

- **`lark-oidc-shim`** (new) — Go HTTP service on k3s, exposed at `auth.app.wash24h.io`. Stateless except for in-memory caches.

- **Lark** (external) — Lark Global (`open.larksuite.com`, `passport.larksuite.com`).

  

### Deployment substrates

  

The identity plane deliberately spans two substrates:

  

- **Incus** (stable core): Keycloak, Postgres 17 on `hcm-db-a`.

- **k3s** (adapter tier): `lark-oidc-shim`, plus future per-IdP shims.

  

This keeps the spine boring and lets adapters benefit from k3s's deployment ergonomics without polluting the IdP host.

  

### Login sequence

  

```

Browser ──► identity.app.wash24h.io/realms/jetx/...

             (Keycloak login page, "Login with Lark" button)

          ──► auth.app.wash24h.io/authorize              [shim]

          ──► open.larksuite.com/open-apis/authen/v1/authorize

          ──► (user authenticates at Lark)

          ──► auth.app.wash24h.io/callback               [shim receives Lark code]

             Shim: fetch app_access_token (cached) →

                   exchange code for user_access_token →

                   call Contacts API →

                   build OIDC claims →

                   sign id_token

          ──► identity.app.wash24h.io/realms/jetx/broker/lark/endpoint

             Keycloak: exchange shim's code at shim /token →

                       verify id_token against shim /jwks.json →

                       JIT-provision or match user by sub=union_id

          ──► originating app with Keycloak's own token

```

  

### Trust boundaries

  

- Browser ↔ shim: TLS (Traefik ACME).

- Browser ↔ Keycloak: TLS (existing).

- Shim → Lark: TLS over the internet, authenticated with `app_id` + `app_secret`.

- Keycloak ↔ shim: TLS, same parent domain (`*.app.wash24h.io`).

  

## 4. API Surface

  

The shim exposes exactly these endpoints on `https://auth.app.wash24h.io`:

  

### Discovery

  

**`GET /.well-known/openid-configuration`** — static JSON:

  

```json

{

  "issuer": "https://auth.app.wash24h.io",

  "authorization_endpoint": "https://auth.app.wash24h.io/authorize",

  "token_endpoint":         "https://auth.app.wash24h.io/token",

  "userinfo_endpoint":      "https://auth.app.wash24h.io/userinfo",

  "jwks_uri":               "https://auth.app.wash24h.io/.well-known/jwks.json",

  "response_types_supported": ["code"],

  "subject_types_supported": ["public"],

  "id_token_signing_alg_values_supported": ["RS256"],

  "scopes_supported": ["openid", "profile", "email"],

  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"]

}

```

  

**`GET /.well-known/jwks.json`** — the shim's public RS256 key in JWKS format (`kid`, `kty`, `n`, `e`).

  

### OIDC authorization code flow

  

**`GET /authorize`** — browser-facing. The browser arrives here via a 302 redirect issued by Keycloak after the user clicks "Login with Lark."

- Accepts: `client_id`, `redirect_uri`, `response_type=code`, `scope`, `state`, `nonce`, `code_challenge`, `code_challenge_method`.

- Validates `client_id` and `redirect_uri` against configured Keycloak broker registration.

- Stashes `{state, nonce, redirect_uri, pkce_challenge}` in an in-memory map keyed by a random `shim_state` (5min TTL).

- Redirects browser to Lark's `/authen/v1/authorize` carrying `shim_state` as Lark's `state` param.

  

**`GET /callback`** — browser-facing. Lark redirects the browser here with `?code=<lark_code>&state=<shim_state>` after the user authenticates at Lark.

- Looks up `shim_state` → recovers original Keycloak params.

- Fetches/reads cached `app_access_token`.

- Exchanges Lark `code` → Lark `user_access_token`.

- Calls Lark `/open-apis/authen/v1/user_info` with `user_access_token`.

- Builds OIDC claim set (Section 5).

- Stores `{claims, nonce, pkce_challenge}` in a second in-memory map keyed by a newly minted shim `code` (60s TTL).

- 302s browser to the original Keycloak `redirect_uri` with `?code=<shim_code>&state=<original_keycloak_state>`.

  

**`POST /token`** — server-to-server from Keycloak.

- Authenticates the caller (`client_secret_basic` or `client_secret_post`).

- Validates `code`, `redirect_uri`, and PKCE `code_verifier`.

- Consumes the code (one-time use).

- Returns:

  ```json

  { "id_token": "<RS256 JWT>", "access_token": "<opaque>", "token_type": "Bearer", "expires_in": 300 }

  ```

  

**`GET /userinfo`** — called with `Authorization: Bearer <access_token>`. Returns the same claim set as the `id_token`, minus `iss`/`aud`/`exp`/`iat`/`nbf`/`jti`/`nonce`.

  

### Operational

  

**`GET /healthz`** — liveness. Returns 200 if the process is up. No Lark call.

  

**`GET /readyz`** — readiness. Returns 200 if the shim has a valid `app_access_token` (or can fetch one). Returns 503 otherwise.

  

## 5. Claim Mapping

  

The shim transforms Lark's Contacts `/open-apis/authen/v1/user_info` response into an OIDC claim set. **This is a cross-app contract** — changing a claim name later breaks Keycloak federation for every existing user.

  

### Tenant invariant

  

Every JetX user with a Lark account has an email (`enterprise_email` or `email`). The shim treats email as **required**.

  

### Standard OIDC claims

  

| OIDC claim | Source (Lark field) | Fallback | Notes |

|---|---|---|---|

| `sub` | `union_id` | — | **Required.** Stable across Lark-app rotation. |

| `email` | `enterprise_email` | `email` | **Required.** If both absent → hard fail (invariant violation, log and error). |

| `email_verified` | `true` | — | Constant. Both Lark email fields are admin-managed. |

| `name` | `name` | — | Full display name in user's preferred locale. Lark's Contacts API returns name as a single string; split `given_name`/`family_name` are not emitted (Keycloak's username mapper handles the single-name case). |

| `preferred_username` | email local-part | — | Derived from `email` (always present). Used as Keycloak username on JIT. |

| `picture` | `avatar_240` | `avatar_url` | 240px avatar. |

| `locale` | `en_name` present ? `en` : `vi` | `en` | Best-effort. |

  

### Lark-specific passthrough claims (`lark_*`)

  

Emitted in both `id_token` and `/userinfo`. Namespaced to avoid future IdP collisions. Not mapped to Keycloak attributes in v1; v2 enables them with Keycloak attribute mappers (no shim redeploy).

  

| Claim | Source | Future use |

|---|---|---|

| `lark_user_id` | `user_id` | Admin-facing employee ID; operator cross-reference. |

| `lark_open_id` | `open_id` | If the shim ever calls other Lark APIs on a user's behalf. |

| `lark_department_ids` | `department_ids` (array) | v2: map to Keycloak groups. |

| `lark_job_title` | `job_title` | v2: Keycloak user attribute. |

| `lark_leader_user_id` | `leader_user_id` | v2: org-chart / approval routing. |

| `lark_employee_no` | `employee_no` | v2: HR cross-linking. |

  

### JWT claims added by the shim itself

  

- `iss` = `https://auth.app.wash24h.io`

- `aud` = `<keycloak_client_id>` (configured)

- `exp` = `iat` + 300s

- `iat` = `now - 10s` (small skew cushion for Keycloak)

- `nbf` = `iat`

- `nonce` = echoed from the original Keycloak `/authorize` request

- `jti` = random UUID v4

  

### Explicitly not emitted

  

- `phone_number` — Lark's `mobile` is opt-in shared; omit to minimize PII surface.

- Group membership — groups assigned inside Keycloak in v1.

  

### Keycloak-side mapping

  

In the `jetx` realm's Lark IdP broker config:

  

- `sub` → federated identity linking (auto).

- `email` → user.email (auto on `email` scope).

- `name`, `picture` → user profile (auto).

- `preferred_username` → user.username on JIT (configure "username mapper: preferred_username").

- `lark_*` → **v2 only.** Leave unmapped in v1.

  

## 6. Deployment & Configuration

  

### Location

  

- Cluster: existing k3s on `wash24h-hcm-a`.

- Namespace: `identity` (new — also hosts future identity-adjacent shims).

- Hostname: `auth.app.wash24h.io`.

- TLS: Traefik built-in ACME.

  

### Manifests

  

All manifests live in `k8s/lark-oidc-shim/` in this repo, applied with `kubectl apply -k`.

  

- `Deployment` — 2 replicas for HA. Each caches its own `app_access_token` independently; redundant Lark calls are within rate limits.

- `Service` — ClusterIP, port 8080.

- `Ingress` — Traefik, TLS for `auth.app.wash24h.io`, routes `/` to the Service.

- `Secret/lark-oidc-shim-secrets` — Lark credentials, Keycloak shared secret, RSA private key.

- `ConfigMap/lark-oidc-shim-config` — non-secret config.

- `ServiceMonitor` or `PodMonitor` (whichever matches the cluster's Prometheus conventions) — scrape `:9090/metrics`.

  

### Container image

  

- Built from this repo's `Dockerfile`. Multi-stage: `golang:1.22` build → `gcr.io/distroless/static:nonroot` runtime.

- Tag: git commit SHA. `latest` is never deployed.

- Registry: **open item — confirm which registry this cluster pulls from** (see §10).

  

### Resource requests & limits

  

- Requests: `50m CPU`, `64Mi memory`.

- Limits: `200m CPU`, `128Mi memory`.

  

### Probes

  

- `livenessProbe`: `GET /healthz`, period 10s, failureThreshold 3 → restart.

- `readinessProbe`: `GET /readyz`, period 5s, failureThreshold 3 → unready.

  

### Ingress sketch

  

```yaml

spec:

  ingressClassName: traefik

  tls:

    - hosts: [auth.app.wash24h.io]

      secretName: auth-app-wash24h-io-tls

  rules:

    - host: auth.app.wash24h.io

      http:

        paths:

          - path: /

            pathType: Prefix

            backend:

              service:

                name: lark-oidc-shim

                port: { number: 8080 }

```

  

Exact ACME annotations match whatever convention `identity.app.wash24h.io` (or other `*.app.wash24h.io` services) already use in the cluster.

  

### Configuration surface (12-factor)

  

All config is read from environment variables.

  

**Secret-backed (K8s Secret `lark-oidc-shim-secrets`):**

  

| Env var | Purpose |

|---|---|

| `LARK_APP_ID` | Lark app identifier from Lark admin console. |

| `LARK_APP_SECRET` | Lark app secret. |

| `KEYCLOAK_CLIENT_SECRET` | Shared secret Keycloak presents at shim `/token`. |

| `JWT_SIGNING_KEY_PEM` | RS256 private key, PEM-encoded. |

  

**ConfigMap-backed (`lark-oidc-shim-config`):**

  

| Env var | Value | Purpose |

|---|---|---|

| `ISSUER_URL` | `https://auth.app.wash24h.io` | Goes into `iss` claim + discovery doc. |

| `LARK_API_BASE` | `https://open.larksuite.com` | Lark Global region. |

| `LARK_PASSPORT_BASE` | `https://passport.larksuite.com` | Token endpoint host. |

| `KEYCLOAK_CLIENT_ID` | `lark-broker` | Client ID Keycloak presents. |

| `KEYCLOAK_REDIRECT_URI` | `https://identity.app.wash24h.io/realms/jetx/broker/lark/endpoint` | Allowlisted redirect for the broker. |

| `ID_TOKEN_TTL_SECONDS` | `300` | 5 minutes. |

| `ACCESS_TOKEN_TTL_SECONDS` | `300` | 5 minutes. |

| `LOG_LEVEL` | `info` | stdlib `slog`. |

  

### One-time secret provisioning (runbook required)

  

1. **RSA keypair** — `openssl genrsa -out signing.pem 2048`, derive `kid` from SHA-256 fingerprint of the DER-encoded public key. Store private half in the Secret. Document procedure in `runbooks/lark-oidc-shim-key-rotation.md`.

2. **Lark app** — create "Custom App" in Lark admin console → Open Platform. Record `app_id` and `app_secret`.

3. **Keycloak broker client secret** — generated when registering the IdP broker in Keycloak. Copy into the Secret.

  

Secret values are **not** committed. Provisioning method (`kubectl create secret`, sealed-secrets, SOPS, or External Secrets Operator) matches the cluster's existing convention — see §10 open items.

  

### Keycloak realm configuration

  

In the `jetx` realm, create an **Identity Provider** of type "OpenID Connect v1.0":

  

- Alias: `lark`

- Discovery URL: `https://auth.app.wash24h.io/.well-known/openid-configuration`

- Client ID: `lark-broker`

- Client secret: matches `KEYCLOAK_CLIENT_SECRET` in the K8s Secret

- Default scopes: `openid profile email`

- Store tokens: **off** (Keycloak does not retain shim tokens)

- Trust email: **on** (shim sets `email_verified=true`)

- Sync mode: `import` (JIT on first login)

  

## 7. State & Persistence

  

The shim is stateless. No database, no Redis, no persistent volume.

  

**In-memory only:**

  

- `app_access_token` cache — one entry per replica, TTL ≈ 2h (Lark's TTL), pre-refreshed at `expires_at − 60s`.

- `shim_state` map — 5min TTL, keyed by random 32-byte values. Entries are evicted on first use.

- `shim_code` map — 60s TTL, keyed by random 32-byte values. One-time use.

  

**Implications:**

  

- Pod restart loses all in-memory state. Cost: one extra `app_access_token` fetch on the next login; in-flight users who hadn't finished authentication start over. Acceptable.

- HA (2 replicas) works without shared cache. Each replica's caches are independent. Lark calls are well within rate limits.

- No refresh tokens issued (Keycloak doesn't need them; skipping them preserves statelessness).

  

## 8. Error Handling

  

### Lark-side failures

  

- **`app_access_token` fetch fails.** `/callback` returns standard OAuth2 `error=server_error` via redirect to the Keycloak `redirect_uri`. Full detail logged server-side; never reflected to user. Readiness flips to 503 after N consecutive failures, Traefik drains.

- **Token race.** Detect Lark "token expired" response, force-refresh once, retry once. No loops.

- **Code exchange fails** (`invalid_grant` from Lark). Propagate `error=invalid_grant` to Keycloak. Log `shim_state` for correlation.

- **Contacts API missing `union_id`.** Hard fail. `sub` is required.

- **Contacts API missing `email` and `enterprise_email`.** Hard fail — tenant invariant violation. Log `email_missing_violation` with `lark_user_id`.

- **Contacts API missing optional fields.** Emit `id_token` without them.

- **Lark rate limit (429).** Return 503 with `Retry-After`. No synchronous retry.

  

### Shim-side state failures

  

- **`shim_state` expired or missing.** Friendly 400 page: "Session expired, please start login again." No redirect (we've lost `redirect_uri`).

- **`shim_code` reuse at `/token`.** `invalid_grant`. Log — Keycloak bug or attack.

- **PKCE mismatch.** `invalid_grant`. Log with `client_id`.

  

### Keycloak-side failures

  

- **Bad `client_secret` at `/token`.** `401 invalid_client`. Log and alert — almost always config drift.

- **Bad `redirect_uri`.** `400 invalid_request`.

- **Keycloak unreachable.** Not the shim's problem; browser sees Keycloak's own error.

  

### Crypto

  

- **Signing key missing at startup.** Fail to start; Kubernetes restarts; operator sees it immediately.

- **Key rotation.** Manual in v1: update Secret, `kubectl rollout restart`. Old in-flight tokens (5min max) expire naturally. Keycloak re-fetches JWKS on signature mismatch.

- **Clock skew.** Accept ±60s on incoming JWT `iat`/`exp`. Emit `iat = now − 10s`.

  

### User-side

  

- **User denies at Lark.** Lark redirects to `/callback` with `error=access_denied`. Shim propagates to Keycloak via OAuth2 error redirect.

- **Frozen / resigned Lark account.** v1: authenticate them anyway (Lark itself blocks OAuth2 for locked accounts). v2: check `status` flags at the shim.

  

### Graceful shutdown

  

- SIGTERM → stop accepting new connections → drain in-flight (max 10s) → exit.

- In-memory state lost; all state is recoverable via re-login or re-fetch.

  

## 9. Observability

  

### Logging

  

- Stdlib `slog` to stdout in JSON format.

- One line per request: `method`, `path`, `status`, `duration_ms`, `shim_state` (if present), `client_id` (if present), `error_code` (if error).

- **No PII in logs.** Never log `union_id`, email, name, or avatar URLs. `lark_user_id` is logged only on invariant-violation errors for operator forensics.

  

### Metrics

  

- Prometheus endpoint on `:9090/metrics` (separate port from `:8080`).

- Counters per endpoint by status code.

- Histogram for Lark API call latency (per upstream endpoint).

- Gauge for `app_access_token` remaining TTL.

  

### Scraping

  

- Prometheus is already scraping Keycloak per platform conventions. Add a scrape target (via `ServiceMonitor` / `PodMonitor` or static scrape config) for the shim.

  

## 10. Testing Strategy

  

### Unit (every commit)

  

- **Claim mapper** (table-driven): happy path, `enterprise_email` vs `email` fallback, both absent → error, missing optional fields, UTF-8 / non-ASCII names, `lark_*` passthrough claims.

- **JWT signing round-trip**: sign with private key → verify via `/.well-known/jwks.json`; assert all expected claims.

- **State maps**: TTL expiry, one-time consumption, eviction.

- **Discovery document**: snapshot vs golden file.

  

### Component (every commit)

  

- `httptest.Server` stubs Lark. Full flow: `/authorize` → stub Lark → `/callback` → shim code → `/token` → JWT verification → `/userinfo`.

- Error paths: Lark 4xx, 5xx, malformed JSON, timeouts. Assert correct OAuth2 error redirects. No panics.

  

### Keycloak integration (PR CI)

  

- `docker-compose.test.yml`: Keycloak + shim + stub Lark.

- Realm config imported from repo. Playwright drives headless browser through the full flow. Asserts a downstream test app receives a valid Keycloak token.

  

### Manual smoke (release gate)

  

Post-staging-deploy checklist:

  

- Fresh login → new Keycloak user JIT-provisioned.

- `sub`, `email`, `name`, `picture` correct in Keycloak admin console.

- `lark_*` attributes present (even if unmapped).

- Re-login matches existing user (no duplicate).

- Logout + re-login works.

- User denies at Lark → Keycloak error page, no 500.

  

### Explicitly not tested

  

- Lark's OAuth2 (not our code).

- Keycloak's broker (not our code).

- Cross-tenant scenarios (single tenant in v1).

- Performance / load (login traffic is trivially bounded).

  

## 11. Open Items (resolve before implementation)

  

- **Image registry.** Which registry does the k3s cluster pull from? (GHCR / self-hosted Harbor / Docker Hub / other.)

- **Secret management pattern.** Plain `Secret`, `sealed-secrets`, SOPS, or External Secrets Operator?

- **Existing cert conventions.** Confirm Traefik built-in ACME is used directly (not cert-manager with a `ClusterIssuer`) for `*.app.wash24h.io` hosts.

- **Prometheus scrape convention.** `ServiceMonitor` (Prometheus Operator) or static scrape config?

- **CLAUDE.md drift.** `CLAUDE.md` currently says Keycloak is at `auth.wash24h.com`. Actual host is `identity.app.wash24h.io`. Update CLAUDE.md separately.

  

## 12. Summary of Decisions (traceable to brainstorm)

  

| # | Decision | Rationale |

|---|---|---|

| Approach | OIDC shim (not SPI, not generic OAuth2 twist) | Boring, isolates Lark knowledge in one place |

| v1 scope | Login only; JIT provisioning; manual groups/roles | Minimum useful increment; defer org sync |

| Q2 | `sub` = `union_id` | Stable across Lark-app rotation; not admin-PII |

| Q3 | Go | Single binary, minimal runtime footprint |

| Q4 | k3s Deployment on `wash24h-hcm-a` | Existing prod orchestrator; stateless fits |

| Q5 | `auth.app.wash24h.io`, Traefik built-in ACME | Per-vendor naming scales; clean ingress |

| Q6 | Baseline OIDC + `lark_*` passthrough claims | Zero future cost; unblocks v2 without shim redeploy |

| Q7 | Lark Global (`open.larksuite.com`) | JetX is Vietnam-based, Global tenant |

| Q8 | 5min id_token, 5min access_token, no refresh | Handshake-only; statelessness |

| Q9 | Fully stateless, in-memory cache only | No DB; RSA keypair in K8s Secret |

  

## 13. Next Steps

  

1. User reviews this spec.

2. Invoke `superpowers:writing-plans` to produce an implementation plan.

3. Resolve §10 open items during planning (or as the first tasks of the plan).