# 01 — Keycloak Setup

> Stand up Keycloak as the identity provider for JetX. Single realm, groups map to teams, one client per application surface.

---

## 1. Deployment

### Container (Incus or Docker)

Keycloak 26.x. PostgreSQL-backed (use the existing `hcm-db-a` server, separate database).

**`infra/keycloak/docker-compose.yml`:**

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    container_name: jetx-keycloak
    restart: unless-stopped
    command: start --optimized
    environment:
      # Database
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://hcm-db-a.tailnet/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD_FILE: /run/secrets/kc_db_password

      # Hostname / proxy
      KC_HOSTNAME: auth.wash24h.com
      KC_HOSTNAME_STRICT: "true"
      KC_PROXY_HEADERS: xforwarded
      KC_HTTP_ENABLED: "true"    # TLS terminated at reverse proxy

      # Admin bootstrap (remove after first login + password change)
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD_FILE: /run/secrets/kc_admin_password

      # Health + metrics
      KC_HEALTH_ENABLED: "true"
      KC_METRICS_ENABLED: "true"

      # Logging
      KC_LOG_LEVEL: INFO
    ports:
      - "127.0.0.1:8080:8080"    # Reverse proxy target; never expose directly
    secrets:
      - kc_db_password
      - kc_admin_password

secrets:
  kc_db_password:
    file: /etc/jetx/secrets/kc_db_password
  kc_admin_password:
    file: /etc/jetx/secrets/kc_admin_password
```

### Database preparation (on `hcm-db-a`)

```sql
CREATE USER keycloak WITH PASSWORD '<strong random>';
CREATE DATABASE keycloak OWNER keycloak;
-- Keycloak requires these on its own schema
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
```

### Reverse proxy (Caddy example)

```caddy
auth.wash24h.com {
    reverse_proxy 127.0.0.1:8080
    encode gzip zstd
}
```

### Prometheus scraping

Keycloak exposes `/metrics` when `KC_METRICS_ENABLED=true`. Add to your Prometheus config:

```yaml
- job_name: keycloak
  metrics_path: /metrics
  static_configs:
    - targets: ['auth.wash24h.com']
  scheme: https
```

---

## 2. Realm configuration

> **Rule:** Do NOT click around in the admin UI as the final step. Configure once, then export the realm to JSON and commit it. Realm JSON is the source of truth.

### Realm: `jetx`

- Login theme: `keycloak` (default) or custom branded later
- Display name: `JetX / Wash24h`
- User registration: **off** (admins create accounts)
- Forgot password: on
- Remember me: off (security)
- Login with email: on
- Duplicate emails: **off**
- SSL required: **external requests**
- Brute force detection: **on**, lock after 5 failures for 15 min

### Tokens

| Setting                         | Value      | Notes                                 |
|---------------------------------|------------|---------------------------------------|
| Access token lifespan           | 5 min      | Short — frontend refreshes silently   |
| Client session idle             | 30 min     | Refresh token max idle                |
| Client session max              | 8 hours    | Hard ceiling                          |
| Offline session idle            | 30 days    | Only for explicitly offline clients   |
| SSO session idle                | 30 min     |                                       |
| SSO session max                 | 10 hours   |                                       |

### Groups (mirror existing teams)

```
/admins
/pi-ops
/staff
/operation-center
```

Assign realm roles to groups, not to users directly. Makes team changes trivial.

### Realm roles

Define coarse roles at the realm level:

- `platform-admin` — full access, break-glass
- `ops` — infrastructure read/write (servers, cameras, Pi nodes)
- `reviewer` — read-only on operational data
- `staff` — default authenticated user

### Client roles (fine-grained, per application)

Defined inside the `jetx-api` client. Use `resource:action` pattern:

- `sites:read`, `sites:write`
- `cameras:read`, `cameras:write`, `cameras:control`
- `users:read`, `users:manage`
- `alpr:read`, `alpr:export`

Map client roles onto realm roles via the **Role mapping** tab. Example:

- `ops` → `sites:*`, `cameras:*`, `alpr:*`
- `reviewer` → `sites:read`, `cameras:read`, `alpr:read`
- `staff` → `sites:read`

### Group → role assignment

| Group              | Realm roles                       |
|--------------------|-----------------------------------|
| `/admins`          | `platform-admin`, `ops`           |
| `/pi-ops`          | `ops`                             |
| `/operation-center`| `reviewer`, `staff`               |
| `/staff`           | `staff`                           |

---

## 3. Clients

### Client: `jetx-web` (SvelteKit frontend)

- **Client type:** OpenID Connect
- **Client authentication:** **On** (confidential) — SvelteKit has a server side
- **Authorization:** Off (we do coarse authz in API, not Keycloak authorization services)
- **Authentication flow:** Standard flow (auth code) + Service accounts **off**
- **Root URL:** `https://app.wash24h.com`
- **Valid redirect URIs:** `https://app.wash24h.com/auth/callback`
- **Valid post-logout redirect URIs:** `https://app.wash24h.com/`
- **Web origins:** `https://app.wash24h.com` (or `+` to reuse redirect URIs)
- **PKCE required:** **S256** (defense in depth; confidential client still benefits)
- **Access token signature:** RS256
- **Client secret:** store in SvelteKit server as `KEYCLOAK_CLIENT_SECRET`

### Client: `jetx-api` (.NET backend)

- **Client type:** OpenID Connect
- **Client authentication:** On (for admin operations only; resource server doesn't need secret to validate tokens)
- **Authorization:** Off
- **Standard flow:** Off
- **Service accounts:** **On** (so .NET can call Keycloak admin API to e.g. list users)
- **Bearer only:** effectively — no redirect URIs needed for token validation
- **Define client roles here** (see above)

### Client scopes

Add a **Group Membership** mapper to the default client scope so group paths appear in tokens:

- Mapper type: Group Membership
- Token claim name: `groups`
- Full group path: **Off** (simpler claims)
- Add to ID token: On, Add to access token: On, Add to userinfo: On

Add a **Realm Role** mapper:

- Token claim name: `realm_access.roles` (default)

Add an **Audience** mapper on `jetx-web`'s client scope to include `jetx-api` in the `aud` claim:

- Included client audience: `jetx-api`
- Add to access token: On

> Without this the .NET API will reject tokens issued to `jetx-web` because audience won't match.

---

## 4. Exporting realm configuration

Once configured, export and commit:

```bash
# From host, exec into container
docker exec -it jetx-keycloak \
  /opt/keycloak/bin/kc.sh export \
  --file /tmp/jetx-realm.json \
  --realm jetx \
  --users realm_file

docker cp jetx-keycloak:/tmp/jetx-realm.json infra/keycloak/realm-export.json
```

Commit `realm-export.json` (sanitize secrets first — replace any client secret with `"$(ENV:CLIENT_SECRET)"` placeholders or strip entirely). This file is your source of truth for realm re-creation / disaster recovery.

### Importing in a fresh environment

```bash
docker run --rm \
  -v $(pwd)/infra/keycloak/realm-export.json:/opt/keycloak/data/import/realm.json \
  -e KC_DB=... \
  quay.io/keycloak/keycloak:26.0 \
  start --import-realm --optimized
```

---

## 5. User creation workflow

Admins create users via Keycloak admin UI or via admin REST API from the .NET backend (service account on `jetx-api` with `manage-users` role in `realm-management` client).

**Minimum user fields:**
- Username (lowercase, email-like: `jimmy.tran`)
- Email (verified: false initially, send verification email)
- First name, last name
- Temporary password: **on** (forces reset)
- Groups: assign to one team group

**Email verification / password reset:** configure SMTP in realm → Email settings. Use a transactional provider (Postmark, SES, SendGrid).

---

## 6. What the token looks like

A successful login for `jimmy.tran` in `/admins` yields an access token with (decoded JWT payload):

```json
{
  "iss": "https://auth.wash24h.com/realms/jetx",
  "aud": ["jetx-api", "account"],
  "sub": "c1e5a8...",
  "azp": "jetx-web",
  "exp": 1713456789,
  "iat": 1713456489,
  "preferred_username": "jimmy.tran",
  "email": "jimmy.tran@wash24h.com",
  "email_verified": true,
  "name": "Jimmy Tran",
  "groups": ["admins"],
  "realm_access": {
    "roles": ["platform-admin", "ops", "staff"]
  },
  "resource_access": {
    "jetx-api": {
      "roles": ["sites:read", "sites:write", "cameras:read", "cameras:write", "cameras:control"]
    }
  }
}
```

The .NET API validates this and maps the `realm_access.roles` + `resource_access.jetx-api.roles` onto ASP.NET Core authorization policies. See `02-dotnet-backend.md`.

---

## 7. Operational checklist

- [ ] Keycloak running behind TLS reverse proxy
- [ ] Bootstrap admin password rotated, stored in password manager
- [ ] Realm `jetx` exported and committed to git (sanitized)
- [ ] SMTP configured for password resets + email verification
- [ ] Brute force detection enabled
- [ ] `/health/ready` endpoint reachable from monitoring
- [ ] `/metrics` scraped by Prometheus
- [ ] PostgreSQL backup covers `keycloak` database (nightly dump)
- [ ] First admin user created, MFA enrolled (TOTP)
- [ ] Break-glass admin account documented, stored offline
