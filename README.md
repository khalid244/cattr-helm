# Cattr Helm Chart

Production-ready Helm chart for deploying [Cattr](https://cattr.app/) time-tracking server on Kubernetes.

Built from real-world deployment experience documented in [cattr-app/server-application#119](https://github.com/cattr-app/server-application/issues/119).

## Quick Start

```bash
helm repo add cattr-helm https://khalid244.github.io/cattr-helm
helm repo update
helm install cattr cattr-helm/cattr
```

## What This Chart Deploys

- **Cattr application** — Laravel-based time-tracking server with s6-overlay
- **MySQL 8.0** — built-in database (no external dependencies required)
- **Persistent storage** — for screenshots and attachments
- **Ingress** — nginx ingress with optional TLS via cert-manager

## Kubernetes Issues Fixed

This chart addresses all known issues from running Cattr on Kubernetes:

| Issue | Fix |
|-------|-----|
| s6-overlay fails with read-only `/var/run` | Mount `/var/run` and `/tmp` as `emptyDir` |
| s6 init timeout during migrations | `S6_CMD_WAIT_FOR_SERVICES_MAXTIME=300000` |
| Cache/session write errors | Mount `/app/storage/framework/cache` and `/app/storage/framework/sessions` as `emptyDir` |
| Log volume issues | `LOG_CHANNEL=stderr` |
| MySQL trigger creation fails | `--log-bin-trust-function-creators=1` |
| App starts before database is ready | Init container waits for MySQL TCP connectivity |
| Migration conflicts (`redmine_instances`, Sanctum) | Documented workarounds below |

## Configuration

### Minimal Production Setup

```bash
helm install cattr cattr-helm/cattr \
  --set cattr.appUrl=https://cattr.example.com \
  --set ingress.hostname=cattr.example.com \
  --set 'ingress.annotations.cert-manager\.io/cluster-issuer=letsencrypt-prod' \
  --set 'ingress.tls[0].secretName=cattr-tls' \
  --set 'ingress.tls[0].hosts[0]=cattr.example.com'
```

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `cattr.appUrl` | Application URL | `http://cattr.local` |
| `cattr.appKey` | Laravel app key (auto-generated if empty) | `""` |
| `cattr.appEnv` | Environment | `production` |
| `cattr.logChannel` | Log channel | `stderr` |
| `database.name` | Database name | `cattr` |
| `database.username` | Database user | `cattr` |
| `database.password` | Database password | `cattr` |
| `database.rootPassword` | MySQL root password | `cattrroot` |
| `mysql.enabled` | Deploy built-in MySQL | `true` |
| `mysql.persistence.size` | MySQL storage size | `10Gi` |
| `persistence.screenshots.size` | Screenshots storage | `5Gi` |
| `persistence.attachments.size` | Attachments storage | `5Gi` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.hostname` | Ingress hostname | `cattr.local` |
| `ingress.className` | Ingress class | `nginx` |

See [values.yaml](charts/cattr/values.yaml) for all available parameters.

### External Database

To use an external MySQL database instead of the built-in one:

```yaml
mysql:
  enabled: false

database:
  host: your-mysql-host.example.com
  port: 3306
  name: cattr
  username: cattr
  password: your-password
```

## Default Credentials

| Field | Value |
|-------|-------|
| Email | `admin@cattr.app` |
| Password | `password` |

Change these immediately after first login.

## Known Migration Issues

On fresh installs, two Cattr migrations may fail due to duplicate table/column creation:

1. **`redmine_instances` table already exists** — the `2020_06_01` migration creates this table, but `2020_07_21` tries to create it again.
2. **`expires_at` column already exists** — Sanctum upgrade migration conflict.

**Fix:** Exec into the MySQL pod and mark them as completed:

```bash
kubectl exec -it <mysql-pod> -- mysql -ucattr -pcattr cattr -e "
  INSERT INTO migrations (migration, batch) VALUES
    ('2020_07_21_095849_create_redmine_instances_table', 1),
    ('2023_03_09_224051_upgrade_laravel_sanctum_to_3_0', 1);
"
```

Then restart the Cattr pod:

```bash
kubectl delete pod -l app.kubernetes.io/component=app
```

## Architecture

```
┌─────────────────────────────────────────────┐
│                  Ingress                     │
│              (nginx + TLS)                   │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│              Cattr Pod                       │
│  ┌────────────────────────────────────────┐  │
│  │  init: wait-for-mysql (busybox)        │  │
│  └────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │  s6-overlay                            │  │
│  │  ├── nginx (port 80)                   │  │
│  │  ├── octane (port 8090)                │  │
│  │  ├── reverb/websocket (port 8080)      │  │
│  │  ├── queue worker                      │  │
│  │  └── supercronic (scheduler)           │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  Volumes:                                    │
│  ├── /var/run          (emptyDir)            │
│  ├── /tmp              (emptyDir)            │
│  ├── /app/storage/framework/cache  (empty)   │
│  ├── /app/storage/framework/sessions (empty) │
│  ├── /app/storage/app/uploads/screenshots    │
│  │                          (PVC, Longhorn)  │
│  └── /app/storage/app/uploads/attachments    │
│                              (PVC, Longhorn) │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│             MySQL Pod                        │
│  mysql:8.0                                   │
│  --log-bin-trust-function-creators=1         │
│  Volume: /var/lib/mysql (PVC, 10Gi)          │
└─────────────────────────────────────────────┘
```

## Differences from the Existing `.helm/` Chart

This chart improves on the existing `.helm/` chart in the Cattr repository:

| Feature | Existing `.helm/` | This chart |
|---------|-------------------|------------|
| Service template | Missing | Included |
| PVC templates | Missing | Included |
| Ingress | Syntax error in YAML | Working, with TLS support |
| APP_KEY generation | Uses non-standard `encryptAES` | Standard `randAlphaNum` + `b64enc` |
| MySQL | Bitnami subchart dependency | Built-in MySQL deployment |
| DB startup ordering | None | Init container waits for MySQL |
| MySQL triggers | Not handled | `--log-bin-trust-function-creators=1` |
| Probes | `php82 artisan octane:status` startup | TCP/HTTP probes |
| Values style | Custom structure | Bitnami-style with `@param` annotations |
| Service account | Not included | Optional, configurable |
| TLS/cert-manager | Not supported | Built-in support |
| `nginx /tmp` | Mounted | Mounted |
| Global StorageClass | Not supported | Supported via `global.storageClass` |

## Requirements

- Kubernetes 1.24+
- Helm 3.x
- nginx-ingress controller (for ingress)
- cert-manager (optional, for TLS)
- Default StorageClass configured

## License

This chart is provided as-is for the Cattr open-source community.
