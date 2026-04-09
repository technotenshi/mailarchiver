# mailarchiver — Tech Stack

## Stack Decisions

| Concern | Choice | Reason |
|---|---|---|
| Ingest tool | **mbsync (isync)** | Fast C binary, native Maildir support, OAuth2 via external password command, actively maintained |
| Custom services language | **Python 3** | Strong stdlib (sqlite3, imaplib), mature IMAP ecosystem, good fit for daemon/worker logic |
| Container orchestration | **Docker Compose v2** | Right complexity level for single-VPS self-hosted deployment; no Kubernetes overhead |
| Scheduling | **Ofelia** | Docker-native job scheduler; triggers container runs on a cron schedule without cron daemons inside containers; `no-overlap: true` on all ingest jobs to prevent concurrent mbsync runs |
| IMAP server | **Dovecot** | Standard, widely supported; serves the Maildir archive to mail clients |
| Backup/sync | **rclone** (crypt overlay) | Mirror-style sync preserves per-file granularity needed by the deletion worker for verification |
| Local backup format | **rsnapshot** on local server | Snapshot-based deduplication over the synced Maildir |
| VPN | **Tailscale** | Zero-config mesh VPN; runs as a container sidecar on the VPS for encrypted sync to local backup |
| At-rest encryption | **gocryptfs + Clevis/Tang** | Directory-level AES-256-GCM on Maildir + manifest DB; passphrase derived via ECDH from Tang server at startup; never stored on VPS disk |
| Tang key server | **Secondary VPS** (lightweight) | Runs Tang daemon; serves JOSE ECDH key-agreement responses; holds no email data; connected via Tailscale |
| Manifest DB | **SQLite** | Sufficient for email volume on a single-server deployment; no external DB process needed |
| Secrets | **Docker secrets** | Native to Docker Compose; age-encrypted files for secrets at rest on disk |
| Base images | **Alpine Linux** | Minimal footprint; mbsync and Python 3 both available via apk |
| Logging | **stdout/stderr → Docker JSON log driver → Promtail → Loki** | Promtail reads Docker JSON log files; no container changes needed |
| Metrics | **Prometheus + Pushgateway** | Daemons expose scrape endpoints; ephemeral containers (mbsync, rclone) push on exit via Pushgateway |
| Dashboards | **Grafana** | Queries both Prometheus (metrics) and Loki (logs) as datasources |
| Alerting | **Alertmanager** | Routes to email (critical) and Slack/Discord webhook (all severities) |

---

## Container Layout

```
┌──────────────────────────────────────────────────────────┐
│                    Docker Compose Stack                   │
│                                                          │
│  ── At startup (host systemd, before Docker) ────────── │
│  gocryptfs-mount.service                                │
│    Clevis ──Tailscale──▶ Tang VPS → ECDH → passphrase  │
│    gocryptfs mounts host paths (maildir, manifest-db)   │
│    docker.service starts only after mount succeeds      │
│  ─────────────────────────────────────────────────────  │
│  ── Application ─────────────────────────────────────── │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│  │  mbsync  │  │ dovecot  │  │  worker  │  │ rclone │  │
│  │(ingest)  │  │ (IMAP)   │  │ (Python) │  │ (sync) │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘  │
│       │              │              │              │      │
│       └──────────────┴──────────────┴──────────────┘     │
│                       Shared Volumes                     │
│                  maildir  /  manifest-db                 │
│                                                          │
│  ┌────────────┐   ┌──────────┐                          │
│  │   ofelia   │   │tailscale │                          │
│  │(scheduler) │   │ (sidecar)│                          │
│  └────────────┘   └──────────┘                          │
│                                                          │
│  ── Observability ───────────────────────────────────── │
│  ┌──────────┐  ┌──────┐  ┌──────────┐  ┌───────────┐  │
│  │prometheus│  │ loki │  │ grafana  │  │alertmanagr│  │
│  └──────────┘  └──────┘  └──────────┘  └───────────┘  │
│  ┌──────────┐  ┌──────────────┐                        │
│  │promtail  │  │ pushgateway  │  ┌───────────────┐     │
│  │(log ship)│  │(ephemeral    │  │ node-exporter │     │
│  └──────────┘  │ job metrics) │  └───────────────┘     │
│                └──────────────┘                         │
└──────────────────────────────────────────────────────────┘
```

### gocryptfs mount strategy

gocryptfs is a FUSE filesystem. FUSE mounts exist in the kernel's mount namespace — a mount inside one Docker container is not visible to another container. Mounting gocryptfs inside a Docker init container and sharing it with other containers requires `privileged: true` or `cap_add: [SYS_ADMIN]` plus explicit mount propagation, which is fragile and widens the privilege surface of the stack.

**Decision: mount on the host before Docker starts.** A systemd unit (`gocryptfs-mount.service`) runs the Clevis+Tang key exchange and mounts the two gocryptfs volumes at host paths (`/var/lib/mailarchiver/maildir` and `/manifest-db`). The unit is ordered `Before=docker.service` — if it fails, Docker does not start. Application containers then bind-mount these host paths as ordinary volumes with no elevated privileges. See `docs/architecture.md §7` for the full startup sequence.

### Container responsibilities

| Container | Lifecycle | Triggered by |
|---|---|---|
| `mbsync` | Ephemeral | Ofelia (e.g. every 15 min) |
| `dovecot` | Long-running daemon | Compose `up` |
| `worker` | Long-running daemon | Compose `up`; polls manifest DB on its own internal schedule |
| `rclone` | Ephemeral | Ofelia (e.g. every hour) |
| `tailscale` | Long-running sidecar | Compose `up` |
| `ofelia` | Long-running scheduler | Compose `up` |
| `prometheus` | Long-running daemon | Compose `up` |
| `loki` | Long-running daemon | Compose `up` |
| `promtail` | Long-running daemon | Compose `up`; mounts Docker socket |
| `grafana` | Long-running daemon | Compose `up` |
| `alertmanager` | Long-running daemon | Compose `up` |
| `pushgateway` | Long-running daemon | Compose `up` |
| `node-exporter` | Long-running daemon | Compose `up` |

### Shared volumes

| Volume | Consumers | Purpose |
|---|---|---|
| `maildir` | mbsync, dovecot, worker, rclone | Central Maildir archive (primary copy) |
| `manifest-db` | mbsync (message registration), worker | SQLite manifest database |

Both volumes are included in the rclone sync scope. `manifest-db` loss without a backup requires full state reconstruction from Maildir scan + rclone check — possible but slow.

### Container restart policy

All long-running containers must declare `restart: unless-stopped`. Without it, a crashed worker, Dovecot, or Alertmanager stays down silently — the `WorkerHalted` alert catches a running-but-halted worker but cannot catch a process that has crashed and not restarted.

### Docker network segmentation

All containers must not share a single flat network. A compromised observability container should not have network access to the Maildir or the deletion worker. Two networks:

```
app-net:   mbsync, dovecot, worker, rclone, tailscale, ofelia, pushgateway
obs-net:   prometheus, loki, grafana, alertmanager, promtail, node-exporter, pushgateway
both:      worker  (exposes /metrics to obs-net; accesses Maildir on app-net)
```

Pushgateway bridges both networks — mbsync/rclone push to it on `app-net`; Prometheus scrapes it on `obs-net`.

### Image digest pinning

All off-the-shelf images must be pinned by digest (`image@sha256:...`), not by mutable tag. Mutable tags like `latest` can be silently updated to a compromised image between deploys. Renovate/Dependabot manages digest updates via PRs through CI.

### Container resource limits

All containers must declare `mem_limit` and `cpus` in Compose to prevent a single runaway process from starving the worker or Dovecot. Specific values to be determined at implementation time based on observed usage.

### Secrets (Docker secrets)

- One secret per provider account: OAuth2 refresh token or app password
- `rclone.conf` — rclone config with crypt remotes configured for B2 and R2
- `age-key` — age private key for unlocking age-encrypted configs at container startup

---

### Secrets additions (observability)

- `smtp-password` — Alertmanager outbound email relay credential
- `slack-webhook-url` — Slack or Discord incoming webhook URL

---

## Open Decisions

- **Grafana dashboard layout** — deferred to implementation phase.
- **Prometheus and Loki retention periods** — default (15 days / low volume) to be tuned based on VPS disk budget.
