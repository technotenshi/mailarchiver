# mailarchiver — Implementation Plan

## Overview

| Iteration | Delivers | Parallel tracks |
|---|---|---|
| 1. Secure Vault + First Ingest | Encrypted Maildir populated on schedule; CI validates from day one | Infrastructure → Bootstrap + Ingest; CI seed runs alongside |
| 2. Readable + Backed Up | Mail client reads archive; 3 backup copies confirmed; key alerts wired | IMAP, Backup pipeline, Core observability, CI extended — all parallel |
| 3. Safe Deletion Live | Deletion worker running with full HALT enforcement; test suite complete | Worker, Backup verifications, Worker observability, CI extended — all parallel |
| 4. Production-Grade | Automated deploy pipeline; system validated against threat model | Full CI/CD, Validation drills, Threat model update — all parallel |

CI/CD and observability are **not separate phases** — they are standing tracks that each iteration extends.

---

## Parallel Track Diagram

```mermaid
Iteration 1 ── "Secure Vault + First Ingest" ────────────────────────────────────
  Track A │ Infrastructure (1.1–1.5)               ← must finish before Track B
  Track B │ Bootstrap + Ingest (2.1–2.3, 3.1–3.4)  ← starts after Track A
  Track C │ CI seed                                 ← starts when any code exists

Iteration 2 ── "Readable + Backed Up" ───────────────────────────────────────────
  Track A │ IMAP service (4.1–4.3)
  Track B │ Backup pipeline (6.1–6.3, 6.5)          ← parallel with Track A
  Track C │ Core observability (7.1–7.2, 7.5–7.6,   ← parallel with Tracks A+B
           │ IngestStale, BackupSyncStale,
           │ BackupMissingFiles alerts)
  Track D │ CI extended (Trivy, promtool, amtool)    ← parallel with Tracks A–C

Iteration 3 ── "Safe Deletion Live" ─────────────────────────────────────────────
  Track A │ Deletion worker (5.1–5.6, TDD)
  Track B │ Backup verifications (6.4, 6.6)          ← parallel with Track A
  Track C │ Worker observability (remaining alerts,   ← parallel with Tracks A+B
           │ Alertmanager routing complete)
  Track D │ CI extended (pytest, mypy, bandit)        ← parallel with Tracks A–C

Iteration 4 ── "Production-Grade" ───────────────────────────────────────────────
  Track A │ Full CI/CD (8.2–8.5)
  Track B │ Validation drills (9.2–9.5)               ← parallel with Track A
  Track C │ Threat model status update (9.6)           ← parallel with Tracks A+B
```

---

## Implementation Conventions

### Test-Driven Development

All custom code (Python worker, OAuth2 helper script, any utility scripts) is written test-first:

1. Write a failing test that specifies the expected behavior
2. Implement the minimum code to make it pass
3. Refactor under passing tests

Tests live in `worker/tests/`. The CI `lint-and-test` job runs `pytest` — no code lands without a passing test suite. Iteration 3 tasks cover the full worker test surface: HALT conditions, state transitions, eligibility logic, and metrics.

### Task Status

| Symbol | Meaning |
|---|---|
| `Pending` | Not started |
| `In Progress` | Actively being worked on |
| `Done` | Acceptance criterion met |

---

## Iteration 1 — Secure Vault + First Ingest

**Goal:** One email lands in the encrypted Maildir on schedule. CI validates code quality from day one.

**Iteration done when:** mbsync cycle completes on Ofelia schedule, email is in the archive, CI is green on push to main.

### Track A — Infrastructure

**Prerequisites:** Two VPS instances provisioned (Ubuntu); Tailscale account.

| # | Task | Done when | Status |
|---|---|---|---|
| 1.1 | Tailscale mesh connects all three nodes (primary VPS, secondary VPS, local server) | `tailscale ping` succeeds between all three nodes | Pending |
| 1.2 | Tang server operational on secondary VPS and reachable only from the primary VPS over Tailscale | From the primary VPS, `curl http://<tang-tailscale-ip>:7500/adv` returns a JOSE JWK response; from a non-primary Tailscale peer, the connection is refused | Pending |
| 1.3 | gocryptfs volumes initialized on primary VPS with Clevis/Tang binding; fallback age passphrase escrowed in password manager | Both cipher directories exist; `clevis decrypt < tang-binding.jwe` succeeds; passphrase stored in password manager | Pending |
| 1.4 | `gocryptfs-mount.service` mounts both volumes automatically at boot (`Before=docker.service`) | After reboot: `mount \| grep gocryptfs` shows both mounts; `systemctl status gocryptfs-mount.service` shows active (exited) | Pending |
| 1.5 | VPS hardening baseline applied | Password SSH auth rejected; fail2ban active; unattended-upgrades enabled; deploy user in `docker` group only | Pending |

See `docs/architecture.md §7` for the full startup sequence and Tang fallback procedure.

### Track B — Container Bootstrap + Email Ingest

**Prerequisites:** Track A complete; Docker Engine + Compose v2 installed on primary VPS; provider accounts and OAuth2 app credentials available.

| # | Task | Done when | Status |
|---|---|---|---|
| 2.1 | Docker Compose file defines all services with correct network assignments (app-net / obs-net) and volume bindings (`maildir`, `manifest-db` bound to Track A host paths) | `docker compose config` exits 0; container network assignments match `docs/tech-stack.md` | Pending |
| 2.2 | Tailscale sidecar container running and authenticated to tailnet | `docker compose ps tailscale` shows running; `docker exec tailscale tailscale status` shows connected | Pending |
| 2.3 | Docker secrets scaffolding in place: all required secrets documented, placeholder mechanism set up, no secrets committed to git | All secrets in `docs/tech-stack.md §Secrets` have a corresponding placeholder; `git status` shows no secret files | Pending |
| 3.1 | `mailarchiver-mbsync` Docker image builds with mbsync binary and OAuth2 XOAUTH2 helper script | `docker build` exits 0; `docker run --rm mailarchiver-mbsync mbsync --version` prints version | Pending |
| 3.2 | OAuth2 refresh token obtained and stored as Docker secret for each Gmail/Outlook account | mbsync OAuth2 helper returns a valid access token for each account | Pending |
| 3.3 | mbsync config syncs each account into its own Maildir namespace (Gmail: `[Gmail]/All Mail` only) | `Maildir/<account>/cur/` contains message files after `docker compose run --rm mbsync` | Pending |
| 3.4 | Ofelia scheduler running with 15-min ingest schedule and `no-overlap: true` on all jobs | `docker compose logs ofelia` shows jobs scheduled; a manually triggered concurrent run is skipped | Pending |

See `docs/architecture.md §5` for OAuth2 device flow details and `docs/tech-stack.md` for the Ofelia `no-overlap` requirement.

### Track C — CI Seed

**Prerequisites:** Any code exists in the repo. Starts in parallel with Track B.

| # | Task | Done when | Status |
|---|---|---|---|
| 8.1a | CI workflow: `lint-and-test` job (ruff check) | ruff passes on push to main | Pending |
| 8.1b | CI workflow: `build-and-scan` job (docker build for worker and mbsync images; no Trivy yet) | Both images build without error in CI | Pending |
| 8.1c | CI workflow: `validate-configs` job (docker compose config, yaml lint) | Compose file validates in CI | Pending |

See `docs/cicd.md` for pipeline design.

---

## Iteration 2 — Readable + Backed Up

**Goal:** Mail client reads the archive over Tailscale. Three backup copies exist and are confirmed. Key ingest and backup alerts are operational.

**Iteration done when:** Thunderbird authenticates to Dovecot and browses mail; `rclone check` exits clean for both B2 and R2; rsnapshot snapshot exists; `IngestStale` and `BackupSyncStale` alerts pass promtool validation; Grafana is secured.

### Track A — IMAP Service (Dovecot)

**Prerequisites:** Iteration 1 complete (Maildir populated).

| # | Task | Done when | Status |
|---|---|---|---|
| 4.1 | Dovecot configured for Maildir storage with IMAPS/993 TLS using a Tailscale-issued certificate | `openssl s_client -connect <host>:993` shows a valid cert; plaintext port 143 refused | Pending |
| 4.2 | Dovecot authentication working with at least one mail user | Mail client (e.g., Thunderbird) authenticates successfully and can browse archived messages | Pending |
| 4.3 | Tailscale ACL restricts Dovecot port 993 to designated mail client devices only | Connection from a `tag:mail-client` peer succeeds; connection from a non-mail-client Tailscale peer is refused | Pending |

See `docs/architecture.md §6` for Dovecot TLS, ACL, and authentication requirements.

### Track B — Backup Pipeline

**Prerequisites:** Iteration 1 complete (Maildir populated). Runs in parallel with Track A.

| # | Task | Done when | Status |
|---|---|---|---|
| 6.1 | B2 and R2 buckets created with object versioning enabled and public access blocked | Versioning confirmed via cloud console; a deleted object is retained as a previous version | Pending |
| 6.2 | rclone crypt remotes configured for B2 and R2; passphrase escrowed in password manager | `rclone lsd b2-crypt:` and `rclone lsd r2-crypt:` succeed; passphrase recorded in password manager | Pending |
| 6.3 | rclone sync job transfers Maildir and manifest-db to B2 and R2 with successful check | `rclone check b2-crypt:maildir /var/lib/mailarchiver/maildir` exits 0 with no missing files | Pending |
| 6.5 | rsnapshot pulling Maildir and manifest-db from primary VPS to local server over Tailscale via `rsync` over SSH on `SSH_PORT` | Snapshot directory on local server contains Maildir files; at least one prior snapshot is retained | Pending |

See `docs/architecture.md §1` for rclone crypt configuration, B2/R2 object versioning requirements, and passphrase escrow.

### Track C — Core Observability

**Prerequisites:** Iteration 1 complete. Runs in parallel with Tracks A and B.

| # | Task | Done when | Status |
|---|---|---|---|
| 7.1 | All 8 observability containers start in Compose (Prometheus, Loki, Grafana, Alertmanager, Alloy, docker-socket-proxy, Pushgateway, node-exporter) | `docker compose ps` shows all 8 containers running; none restart-looping | Pending |
| 7.2 | Alloy ships container logs to Loki with correct labels through the socket proxy | Grafana Explore → Loki query for `{container="worker"}` returns recent log lines, and Alloy has no direct `/var/run/docker.sock` bind mount | Pending |
| 7.5 | Grafana secured: no anonymous access, admin password via Docker secret, not exposed on public VPS IP, and reachable only from `tag:admin-device` over Tailscale | Anonymous request to Grafana returns 401; Grafana port is unreachable from the public internet and from non-admin Tailscale peers | Pending |
| 7.6 | healthchecks.io checks created and receiving heartbeats from worker and Ofelia | healthchecks.io dashboard shows both checks green after one full worker cycle and one mbsync run | Pending |
| 7.3a | Alert rules for ingest and backup: `IngestStale`, `BackupSyncStale`, `BackupMissingFiles`, `MaildirDiskPressure`, `VPSUnreachable` | `promtool check rules config/prometheus/alerts.yml` exits 0; all five alert names present | Pending |
| 7.4a | Alertmanager routing skeleton: Slack/Discord webhook receives test alert for Warning severity | `amtool check-config` exits 0; manually triggered test alert appears in Slack channel | Pending |

See `docs/observability.md` for the full alert mapping, routing rules, and access control requirements.

### Track D — CI Extended

**Prerequisites:** Iteration 1 Track C (CI seed) passing. Runs in parallel with Tracks A–C.

| # | Task | Done when | Status |
|---|---|---|---|
| 8.1d | CI `build-and-scan`: add Trivy scan for all off-the-shelf images (dovecot, rclone, prometheus, loki, grafana, alertmanager, docker-socket-proxy) pulled by their pinned digest | Trivy scan passes on CI; HIGH/CRITICAL CVEs fail the job | Pending |
| 8.1e | CI `validate-configs`: add promtool check config and amtool check-config steps | Both tools validate cleanly in CI; alert configs must pass before merge | Pending |

See `docs/cicd.md §build-and-scan` and `docs/cicd.md §validate-configs`.

---

## Iteration 3 — Safe Deletion Live

**Goal:** Deletion worker runs full cycles with all HALT conditions enforced. All worker logic is covered by tests. Full CI lint-and-test suite is green.

**Iteration done when:** Worker completes a live cycle; a message eligible on all three conditions is deleted from its provider account (9.1); all four HALT conditions have passing unit tests; Alertmanager routes critical alerts to email.

### Track A — Deletion Worker

**Prerequisites:** Iteration 1 complete; Maildir populated. All worker code written test-first per TDD conventions.

| # | Task | Done when | Status |
|---|---|---|---|
| 5.1 | SQLite schema deployed with WAL mode enabled | All 4 tables exist; `PRAGMA journal_mode` returns `wal`; `PRAGMA integrity_check` returns `ok` | Pending |
| 5.2 | mbsync writes `messages` and `provider_copies` rows to manifest DB after each ingest run | `SELECT COUNT(*) FROM messages` increases after an mbsync run | Pending |
| 5.3 | Worker daemon implements full cycle logic: startup PENDING reset, eligibility query, per-account IMAP EXPUNGE, state transitions (PENDING → DELETED / ELIGIBLE + failure_count) | Worker starts without error; dry-run mode logs eligible candidates without deleting; cycle completes and worker sleeps | Pending |
| 5.4 | Worker exposes all four Prometheus metrics on `/metrics` endpoint | `curl worker:8000/metrics` returns `mailarchiver_worker_halt`, `worker_eligible_total`, `worker_deletions_total`, `backup_last_verified_timestamp_seconds` | Pending |
| 5.5 | All four HALT conditions covered by unit tests | `pytest tests/` passes; coverage includes DB integrity failure, DB inaccessible, all backups MISSING 3+ cycles, and 5 consecutive auth failures per account | Pending |
| 5.6 | Worker container image builds and starts as a long-running daemon | `docker compose up -d worker` succeeds; logs show cycle started; container stays running | Pending |

See `docs/architecture.md §2` for the full schema, worker algorithm pseudocode, and HALT condition table.

### Track B — Backup Verifications

**Prerequisites:** Iteration 2 Track B complete (rclone and rsnapshot running); Track A task 5.1 complete (schema deployed). Runs in parallel with Track A.

| # | Task | Done when | Status |
|---|---|---|---|
| 6.4 | `backup_verifications` rows updated to CONFIRMED for B2 and R2 after each rclone check | SQL query shows CONFIRMED rows for both B2 and R2 destinations after a rclone run | Pending |
| 6.6 | `backup_verifications` LOCAL rows updated to CONFIRMED after each rsnapshot run | SQL query shows CONFIRMED row for LOCAL destination after a snapshot run | Pending |

### Track C — Worker Observability

**Prerequisites:** Iteration 2 Track C complete (Prometheus, Loki, Alertmanager skeleton running). Runs in parallel with Tracks A and B.

| # | Task | Done when | Status |
|---|---|---|---|
| 7.3b | Remaining alert rules: `WorkerHalted`, `ManifestIntegrityFailure`, `DeletionFailure`, `IntegrityError`, `AuthFailure` | `promtool check rules` exits 0; all 10 alert names from `docs/observability.md` present | Pending |
| 7.4b | Alertmanager routing complete: critical alerts (`WorkerHalted`, `IntegrityError`, `MaildirDiskPressure`, `VPSUnreachable`) routed to email + Slack | Test critical alert delivered to email; all severity levels reach Slack | Pending |

### Track D — CI Extended

**Prerequisites:** Iteration 2 Track D passing. Runs in parallel with Tracks A–C.

| # | Task | Done when | Status |
|---|---|---|---|
| 8.1f | CI `lint-and-test`: add mypy type check, bandit security scan, pytest with full worker test suite | All three tools pass in CI; HALT condition and state-transition tests included | Pending |

---

## Iteration 4 — Production-Grade

**Goal:** Automated deploy pipeline. System validated against the threat model. Ready for long-term unattended operation.

**Iteration done when:** `v0.1.0` tag triggers automated deploy and webhook fires; DR drill completes on a clean VPS within documented RTO; all T1–T18 threat model entries reflect implemented state.

### Track A — Full CI/CD Pipeline

**Prerequisites:** Iteration 3 complete (all code, tests, and configs fully exist).

| # | Task | Done when | Status |
|---|---|---|---|
| 8.2 | Release workflow builds and pushes worker and mbsync images to ghcr.io on `v*` tag | `v0.1.0` tag → both images appear in ghcr.io with semver, SHA, and latest tags | Pending |
| 8.3 | Deploy job SSHs to VPS, runs `docker compose pull && up -d`, health-checks, and notifies webhook | Slack deploy notification received; `docker compose ps` on VPS shows all containers running | Pending |
| 8.4 | All Actions references pinned by commit SHA; all Compose image references pinned by digest | No `@v4`-style or `:latest` references in workflow files or docker-compose.yml | Pending |
| 8.5 | Branch protection configured on `main` and Renovate opens first dependency update PR | Direct push to main rejected; Renovate opens a PR for a stale image digest or Python dep | Pending |

See `docs/cicd.md` for pipeline design, security requirements, and rollback procedure.

### Track B — Validation Drills

**Prerequisites:** Full system running (all Iterations 1–3 complete). Runs in parallel with Track A.

| # | Task | Done when | Status |
|---|---|---|---|
| 9.2 | Tang failure resilience confirmed: Tang unreachable → primary VPS does not boot into Docker | After stopping Tang and rebooting primary VPS: `gocryptfs-mount.service` exits non-zero; `docker ps` shows no containers | Pending |
| 9.3 | Worker HALT confirmed: manifest DB corruption → worker halts and alert fires | After injecting DB corruption: worker logs `db_integrity_check_failed`; `worker_halt` gauge = 1; `ManifestIntegrityFailure` alert in Alertmanager | Pending |
| 9.4 | Backup gap blocks deletion: removing one B2 object → worker skips that message's deletion | After deleting one B2 object: next worker cycle leaves that message's `deletion_status = ELIGIBLE`; `BackupMissingFiles` alert fires | Pending |
| 9.5 | Disaster recovery drill succeeds on a clean test VPS following `docs/recovery.md` | Mail client can read archived messages on the recovered VPS; recovery completed within documented RTO | Pending |

### Track C — Threat Model Status Update

**Prerequisites:** Validation drills (Track B) complete. Runs in parallel with Track A.

| # | Task | Done when | Status |
|---|---|---|---|
| 9.6 | Threat model statuses updated to reflect implemented state | All 18 entries in `docs/threat-model.md` show `Implemented` or an accurately scoped status | Pending |

---

## Repo Structure at Completion

```
mailarchiver/
├── docker-compose.yml
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── worker/
│   ├── Dockerfile
│   ├── main.py
│   ├── db.py            (schema + queries)
│   ├── imap.py          (IMAP session handler)
│   ├── metrics.py       (Prometheus endpoint)
│   └── tests/
├── mbsync/
│   ├── Dockerfile
│   ├── mbsyncrc         (template; real config from Docker secret)
│   └── oauth2-helper.sh
├── config/
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── alerts.yml
│   ├── alertmanager/
│   │   └── alertmanager.yml
│   ├── loki/
│   ├── promtail/
│   ├── grafana/
│   └── ofelia.ini
├── systemd/
│   └── gocryptfs-mount.service
└── docs/
```
