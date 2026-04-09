# mailarchiver

A self-hosted system that permanently archives email from multiple providers (Gmail, Outlook, Yahoo) into a central Maildir store, maintains three encrypted backup copies, and only removes provider-side copies after verifying retention and backup conditions are met.

## The problem

Email stored only at a provider is not truly under your control. Providers can lose data, terminate accounts, or be compelled to delete content. This system creates a permanent, encrypted, self-owned archive — while keeping the provider copies alive until you're certain the backups are solid.

## How it works

```
Gmail ─┐
Outlook─┤─(IMAP pull)─▶ Primary Maildir ──▶ Dovecot (IMAP) ──▶ Mail clients
Yahoo ─┘     (mbsync)    (VPS, encrypted)
                              │
                    ┌─────────┼──────────────┐
                    ▼         ▼              ▼
              Local rsnapshot  Backblaze B2  Cloudflare R2
              (Copy #2)       (Copy #3,     (Copy #4,
                               encrypted)    encrypted)
                    └─────────┴──────────────┘
                                   │
                           Deletion worker
                    (deletes provider copy only after
                     90 days archived + all 3 backups confirmed)
```

The **deletion worker** is the core safety mechanism. It never deletes an email from a provider account until:
1. The message has been in the primary archive for at least 90 days
2. All three backup copies have been confirmed within the last 7 days
3. The manifest database shows no integrity errors

If any condition fails, the provider copy is preserved.

## Architecture

- **Ingest** — mbsync (isync) via IMAP; OAuth2 for Gmail/Outlook, app passwords for others
- **Primary archive** — Maildir on a VPS, encrypted at rest with gocryptfs (passphrase derived at startup via Clevis + Tang on a secondary VPS)
- **Backups** — Local rsnapshot over Tailscale VPN; Backblaze B2 and Cloudflare R2 via rclone crypt (AES-256)
- **IMAP service** — Dovecot, accessible only to Tailscale peers
- **Deletion worker** — Python daemon with SQLite manifest database
- **Observability** — Prometheus + Loki + Grafana + Alertmanager; external heartbeat via healthchecks.io
- **Orchestration** — Docker Compose v2 on a single VPS

## Status

**Design complete — implementation not yet started.**

All architectural decisions are documented in `docs/`. The next phase is implementation.

## Documentation

| Doc | Covers |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Full system design: encryption, deletion worker algorithm, retention policy, resilience, gocryptfs startup sequence |
| [docs/tech-stack.md](docs/tech-stack.md) | Container stack, Docker Compose layout, network segmentation |
| [docs/observability.md](docs/observability.md) | Metrics, alerts, dashboards, external heartbeat |
| [docs/cicd.md](docs/cicd.md) | GitHub Actions pipelines, image publishing, deploy procedure |
| [docs/threat-model.md](docs/threat-model.md) | 14 threats with mitigations and security gaps |
| [docs/recovery.md](docs/recovery.md) | Disaster recovery procedure, RPO/RTO |
| [docs/operations.md](docs/operations.md) | Adding accounts, rotating credentials, Tang key rotation |

## Requirements (planned deployment)

- Two VPS instances: primary (Ubuntu, Docker) + secondary (Tang key server, minimal)
- Local server with rsnapshot for local backup
- Tailscale account (connects all three nodes)
- Backblaze B2 and Cloudflare R2 bucket credentials (bucket-scoped API keys)
- Gmail and/or Outlook OAuth2 app credentials (device authorization flow)
- healthchecks.io account (external heartbeat)
