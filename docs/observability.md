# mailarchiver — Observability

## Stack Decision

**Prometheus + Loki + Grafana + Alertmanager + Alloy + Docker Socket Proxy + Pushgateway + Node Exporter**

All Grafana ecosystem, all Docker Compose–native, all lightweight.

### Why not ELK

Elasticsearch requires 2–4 GB RAM minimum (JVM). On a single VPS, that exceeds the resource footprint of the mail archiver itself. ELK is also log-centric; metrics require Metricbeat on top. Overkill for this volume.

### Why not OpenTelemetry

OTel is an instrumentation SDK and collection standard — it still requires a backend (Prometheus, Jaeger, etc.). Its value is vendor-neutral portability across polyglot microservices. This system has one custom service, no cross-service traces, and no backend migration planned. The SDK overhead is not justified.

### Why this stack

- **Loki** is order-of-magnitude lighter than Elasticsearch — it indexes log metadata (labels), not full text. A typical instance runs ~200 MB RAM.
- **Alloy** (Grafana Alloy, the supported successor to Promtail) ships logs from Docker's JSON log files with zero changes to application containers.
- **Pushgateway** solves the ephemeral container problem: mbsync and rclone push metrics on exit; Prometheus scrapes on its normal interval. This is the canonical pattern for batch/cron job observability.
- **Grafana** queries both Prometheus and Loki as datasources — metrics and logs are correlated in the same dashboards.
- All five failure modes from `architecture.md §4` are covered by this stack without custom log-scraping scripts.

---

## Components Added to Docker Compose

| Container | Role | Lifecycle |
|---|---|---|
| `prometheus` | Metrics store + alerting rule evaluation | Long-running |
| `loki` | Log store (label-indexed, not full-text) | Long-running |
| `alloy` | Log shipper — reads Docker JSON log files → Loki | Long-running |
| `docker-socket-proxy` | Read-only Docker API allowlist for Alloy metadata discovery | Long-running |
| `grafana` | Dashboards; queries Prometheus and Loki | Long-running |
| `alertmanager` | Alert routing to email and Slack/Discord webhook | Long-running |
| `pushgateway` | Receives push metrics from ephemeral containers | Long-running |
| `node-exporter` | Host metrics: disk, CPU, memory | Long-running |

All on Alpine base images. No additional volumes beyond standard Prometheus/Loki data directories.

---

## Instrumentation Plan

### Ephemeral containers → Pushgateway (push on exit)

mbsync and rclone push a small set of metrics to Pushgateway when their container exits. Prometheus scrapes Pushgateway on its interval and retains the last-pushed value until overwritten.

**mbsync (per account):**
```
mailarchiver_ingest_messages_fetched_total{account}
mailarchiver_ingest_duration_seconds{account}
mailarchiver_ingest_last_success_timestamp_seconds{account}
mailarchiver_ingest_exit_code{account}
```

**rclone (per destination: LOCAL, B2, R2):**
```
mailarchiver_sync_files_transferred_total{destination}
mailarchiver_sync_missing_files_total{destination}
mailarchiver_sync_last_success_timestamp_seconds{destination}
mailarchiver_sync_duration_seconds{destination}
```

### Python worker → Prometheus scrape endpoint (long-running)

The worker exposes a lightweight HTTP metrics endpoint (e.g. `/metrics`) scraped by Prometheus on its normal interval.

```
mailarchiver_worker_halt                                  # gauge: 1 if halted, 0 if running
mailarchiver_worker_eligible_total{account}               # messages eligible for deletion
mailarchiver_worker_deletions_total{account, status}      # status: success | failed | retrying
mailarchiver_backup_last_verified_timestamp_seconds{destination}
```

### Logs → Alloy → Loki (zero application-container changes)

Alloy reads JSON log files written by Docker's logging driver for all containers. For container discovery and labels, it queries Docker metadata through an internal `docker-socket-proxy` service on `obs-net`. No application sidecar and no application-container log changes are needed.

- Python worker emits structured JSON log lines (`level`, `event`, `account`, `message_id`, `action`)
- mbsync and rclone write to stdout/stderr (captured as-is)
- Dovecot writes to syslog (Alloy reads via Docker log capture)

The socket proxy enforces a read-only Docker API allowlist; see `docs/tech-stack.md §Docker socket proxy` for the full policy. Mutating paths remain disabled.

Labels applied by Alloy: `container`, `account` (parsed from structured lines where present).

### Node Exporter → Prometheus scrape

Provides host-level metrics. Key signal: disk usage on the `maildir` volume.

### External heartbeat → healthchecks.io

Prometheus and Alertmanager run on the same VPS as the application. A total VPS outage takes down the entire observability stack — no alerts fire, the operator receives silence. This cannot be solved from within the stack.

The worker pushes a heartbeat ping to an external service (healthchecks.io or equivalent) after each successful poll cycle. Ofelia also pings a separate check after each mbsync run completes. If either heartbeat goes missing for more than the check interval, the external service sends the notification independently of the VPS.

This is the only mechanism that can detect a total VPS outage.

---

## Alert Mapping

Every failure mode from `architecture.md §4` maps to an alert:

| Failure mode | Alert name | Signal type | Condition |
|---|---|---|---|
| Backup destination unreachable | `BackupSyncStale` | Metric | `sync_last_success_timestamp` > 2h ago, per destination |
| rclone check finds missing file | `BackupMissingFiles` | Metric | `sync_missing_files_total > 0` for any destination |
| Primary archive disk full | `MaildirDiskPressure` | Metric | Node Exporter maildir volume > 80% |
| Primary file missing despite confirmed backup | `IntegrityError` | Log | Loki alert on worker log: `level=critical event=integrity_error` |
| Provider IMAP deletion fails | `DeletionFailure` | Metric | `worker_deletions_total{status="failed"} > 0` |
| Deletion worker halted | `WorkerHalted` | Metric | `worker_halt == 1` |

Additional:

| Alert | Signal | Condition |
|---|---|---|
| `IngestStale` | Metric | `ingest_last_success_timestamp` > 30 min ago, per account |
| `AuthFailure` | Log | Loki alert on mbsync log: auth error patterns |
| `ManifestIntegrityFailure` | Log | Loki alert on worker log: `event=db_integrity_check_failed` |
| `VPSUnreachable` | External | healthchecks.io: heartbeat missing > grace period (external to VPS) |

---

## Alert Routing

Alertmanager routes to two channels:

| Severity | Email | Slack / Discord webhook |
|---|---|---|
| Critical (`WorkerHalted`, `IntegrityError`, `MaildirDiskPressure`, `VPSUnreachable`) | Yes | Yes |
| Warning (`BackupSyncStale`, `BackupMissingFiles`, `DeletionFailure`, `ManifestIntegrityFailure`) | No | Yes |
| Info (`IngestStale`, `AuthFailure`) | No | Yes (low-priority channel or thread) |

Credentials (SMTP password, webhook URL) stored as Docker secrets. Never in config files.

---

## Access Controls

### Grafana

Grafana must not be exposed anonymously. Email metadata (account names, message counts, backup status, deletion history) reveals personal correspondence patterns.

- Anonymous access disabled (`allow_sign_up = false`)
- Admin password set via Docker secret (not the default `admin/admin`)
- Grafana accessible only within Tailscale network — not on the public VPS IP
- Grafana reachable only from Tailscale peers tagged `tag:admin-device`
- Consistent with Dovecot: all operator-facing services scoped to explicitly authorized Tailscale peers

### Internal observability services

Prometheus (9090), Loki (3100), Alertmanager (9093), and Pushgateway (9091) are on the Docker internal network only. They must not be port-mapped to `0.0.0.0` on the VPS host. Grafana is the single ingress point for operator access.

---

## Open Decisions

- **Grafana dashboard layout** — deferred; defined when implementation begins.
- **Prometheus retention period** — default 15 days is likely sufficient; tune based on VPS disk budget.
- **Loki retention period** — similarly deferred; indexed metadata is small, raw log volume is low.
