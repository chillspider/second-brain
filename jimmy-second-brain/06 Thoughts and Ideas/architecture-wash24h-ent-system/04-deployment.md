# 04 — Deployment

> Production deployment on existing JetX infrastructure: Incus containers, systemd, Caddy/nginx, Prometheus. No Kubernetes for this stack — not worth the complexity for three services.

---

## 1. Topology

```
┌─────────────────────── wash24h-hcm-a ──────────────────────┐
│                                                            │
│  Caddy (TLS, reverse proxy)  :443                          │
│    ├─ auth.wash24h.com  ──►  jetx-keycloak (Incus)  :8080  │
│    ├─ api.wash24h.com   ──►  jetx-api (systemd)     :5000  │
│    └─ app.wash24h.com   ──►  jetx-web (systemd)     :3000  │
│                                                            │
│  jetx-keycloak  (Incus container)                          │
│  jetx-api       (systemd service, .NET 9 runtime)          │
│  jetx-web       (systemd service, Node 22 + SvelteKit)     │
│                                                            │
└────────────────────────────────────────────────────────────┘
                           │
                           ▼
              hcm-db-a (PostgreSQL 17)
              ├─ keycloak   database
              └─ jetx_app   database
```

Services talk to each other over `127.0.0.1` or Tailscale for cross-host.

---

## 2. Directory layout on host

```
/opt/jetx/
├── api/                   # .NET API publish output
│   ├── JetX.Api           # binary
│   ├── *.dll
│   └── appsettings.json
├── web/                   # SvelteKit build output
│   ├── build/
│   ├── package.json
│   └── node_modules/
└── migrator/              # dotnet ef migrations bundle
    └── efbundle

/etc/jetx/
├── api.env                # root:jetx 640
├── web.env                # root:jetx 640
├── secrets/
│   ├── kc_db_password     # root:root 600
│   └── kc_admin_password  # root:root 600
└── caddy/Caddyfile

/var/log/jetx/             # logs if not going to journald (prefer journald)
```

Create a `jetx` system user:

```bash
useradd --system --shell /usr/sbin/nologin --home /opt/jetx jetx
chown -R jetx:jetx /opt/jetx
chmod 750 /etc/jetx
chown root:jetx /etc/jetx
```

---

## 3. systemd units

### `/etc/systemd/system/jetx-api.service`

```ini
[Unit]
Description=JetX API
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=jetx
Group=jetx
WorkingDirectory=/opt/jetx/api
EnvironmentFile=/etc/jetx/api.env

# Run migrations before the API comes up
ExecStartPre=/opt/jetx/migrator/efbundle --connection "${ConnectionStrings__AppDb}"

ExecStart=/usr/bin/dotnet /opt/jetx/api/JetX.Api.dll

Restart=on-failure
RestartSec=5

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/jetx
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=false   # .NET JIT needs W|X
RestrictRealtime=true
SystemCallArchitectures=native
CapabilityBoundingSet=

# Logging to journald
StandardOutput=journal
StandardError=journal
SyslogIdentifier=jetx-api

[Install]
WantedBy=multi-user.target
```

### `/etc/systemd/system/jetx-web.service`

```ini
[Unit]
Description=JetX Web (SvelteKit)
After=network-online.target jetx-api.service
Wants=network-online.target

[Service]
Type=simple
User=jetx
Group=jetx
WorkingDirectory=/opt/jetx/web
EnvironmentFile=/etc/jetx/web.env
Environment=NODE_ENV=production PORT=3000 HOST=127.0.0.1

ExecStart=/usr/bin/node build

Restart=on-failure
RestartSec=5

NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=true
RestrictRealtime=true
SystemCallArchitectures=native
CapabilityBoundingSet=

StandardOutput=journal
StandardError=journal
SyslogIdentifier=jetx-web

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
systemctl daemon-reload
systemctl enable --now jetx-api jetx-web
```

---

## 4. Keycloak on Incus

```bash
incus launch images:debian/12 jetx-keycloak
incus config device add jetx-keycloak kcport proxy \
    listen=tcp:127.0.0.1:8080 connect=tcp:127.0.0.1:8080
incus exec jetx-keycloak -- bash -c "apt-get update && apt-get install -y docker.io"
incus exec jetx-keycloak -- docker compose -f /root/keycloak/docker-compose.yml up -d
```

See `01-keycloak-setup.md` for the compose file.

---

## 5. Caddy reverse proxy

**`/etc/jetx/caddy/Caddyfile`:**

```caddy
{
    email ops@wash24h.com
}

auth.wash24h.com {
    reverse_proxy 127.0.0.1:8080
    encode gzip zstd
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}

api.wash24h.com {
    reverse_proxy 127.0.0.1:5000
    encode gzip zstd
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
    }
    # Tight rate limit on login-adjacent paths if needed
}

app.wash24h.com {
    reverse_proxy 127.0.0.1:3000
    encode gzip zstd
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        Content-Security-Policy "default-src 'self'; connect-src 'self' https://auth.wash24h.com; img-src 'self' data: blob:; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    }
}
```

Let's Encrypt certs are automatic. For internal-only deployment, switch to `tls internal` with Caddy's internal CA or use your Tailscale hostname with `tls { on_demand }`.

---

## 6. Build & deploy pipeline

Until you have CI, a deploy script on your workstation is fine.

**`scripts/deploy.sh`:**

```bash
#!/usr/bin/env bash
set -euo pipefail

HOST="wash24h-hcm-a"
REMOTE="/opt/jetx"

# Backend
pushd backend
dotnet publish src/JetX.Api -c Release -o ../build/api --no-self-contained
dotnet ef migrations bundle \
    --project src/JetX.Infrastructure \
    --startup-project src/JetX.Api \
    -o ../build/migrator/efbundle --force
popd

# Frontend
pushd frontend
npm ci
npm run build
popd

# Sync
rsync -avz --delete build/api/       "$HOST:$REMOTE/api/"
rsync -avz --delete build/migrator/  "$HOST:$REMOTE/migrator/"
rsync -avz --delete frontend/build/  "$HOST:$REMOTE/web/build/"
rsync -avz frontend/package.json frontend/package-lock.json "$HOST:$REMOTE/web/"

ssh "$HOST" "cd $REMOTE/web && npm ci --omit=dev"
ssh "$HOST" "systemctl restart jetx-api jetx-web"
ssh "$HOST" "systemctl status --no-pager jetx-api jetx-web"
```

---

## 7. Monitoring

### Prometheus scrape targets

```yaml
- job_name: jetx-api
  metrics_path: /metrics
  static_configs:
    - targets: ['api.wash24h.com']
  scheme: https

- job_name: keycloak
  metrics_path: /metrics
  static_configs:
    - targets: ['auth.wash24h.com']
  scheme: https

- job_name: node-jetx-host
  static_configs:
    - targets: ['wash24h-hcm-a:9100']
```

### Key metrics to alert on

| Metric                                         | Threshold        | Meaning                 |
|------------------------------------------------|------------------|-------------------------|
| `http_server_request_duration_seconds{p=0.95}` | > 1s for 5m      | API slow                |
| `dotnet_gc_collections_total`                  | sharp increase   | Memory pressure         |
| `keycloak_user_event{event="LOGIN_ERROR"}`     | > 20/min         | Brute force or breakage |
| `process_resident_memory_bytes` (api/web)      | > 1GB            | Leak suspect            |
| Cert expiry                                    | < 7 days         | Caddy usually handles   |

### Logs

All three services log to journald. Ship to Loki or keep local with `journalctl` retention:

```ini
# /etc/systemd/journald.conf.d/jetx.conf
[Journal]
SystemMaxUse=2G
MaxRetentionSec=30day
```

---

## 8. Backups

**PostgreSQL (hcm-db-a):** nightly `pg_dump` of both `keycloak` and `jetx_app` to MinIO.

```bash
#!/usr/bin/env bash
# /usr/local/bin/jetx-backup.sh (runs via systemd timer on hcm-db-a)
set -euo pipefail
TS=$(date +%Y%m%d-%H%M%S)
for DB in keycloak jetx_app; do
    pg_dump -Fc -U postgres "$DB" | \
        mc pipe "minio/jetx-backups/postgres/$DB/$DB-$TS.dump"
done
```

**Keycloak realm JSON:** re-export after any significant config change and commit.

**Secrets:** never in backups. Stored separately in a password manager / offline.

---

## 9. Runbook cheatsheet

| Task                                  | Command                                                               |
|---------------------------------------|-----------------------------------------------------------------------|
| Restart API                           | `systemctl restart jetx-api`                                          |
| Tail API logs                         | `journalctl -u jetx-api -f`                                           |
| Apply DB migration manually           | `sudo -u jetx /opt/jetx/migrator/efbundle --connection "$CONN"`       |
| Enter Keycloak container              | `incus exec jetx-keycloak -- bash`                                    |
| Force Keycloak realm re-import        | See `01-keycloak-setup.md` §4                                         |
| Rotate Keycloak client secret         | Admin UI → clients → jetx-web → credentials → regenerate; update env  |
| Reload Caddy                          | `systemctl reload caddy`                                              |
| Check which node a Pi calls home to   | `journalctl -u wireguard@wg0 -f` (if migrated off Tailscale)          |
