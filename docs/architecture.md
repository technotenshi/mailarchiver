# mailarchiver — High-Level Architecture

## 1. Encryption Strategy (B2 / R2 offsite)

**Decision: rclone crypt as the encryption layer**

All offsite transfers pass through `rclone crypt`, which encrypts file names and content before bytes leave the server. The Maildir directory structure is preserved logically but is opaque to the storage provider.

- One rclone remote per destination (B2, R2), each with its own crypt overlay
- **B2 and R2 API keys must be scoped to the specific backup bucket only** — not account-level; a leaked API key must not grant access to other buckets or account management
- **Block public access on both buckets**; no pre-signed URL generation permitted
- A single encryption passphrase (and salt) stored in the server's secret store (e.g. Vault, or a local `age`-encrypted file on the VPS)
- Encryption is AES-256-CTR with Poly1305 MAC (rclone's default)
- The local Maildir archive is encrypted at rest using gocryptfs — see §7

**Passphrase escrow (critical):** The rclone crypt passphrase is the single key that makes all offsite backups recoverable. If the VPS is destroyed and the passphrase is lost with it, B2 and R2 become permanently unrecoverable. The passphrase must be stored in at least one location independent of the VPS — a personal password manager, a printed copy in a secure location, or a second encrypted copy in a geographically separate secret store.

**B2 and R2 object versioning:** rclone `sync` makes the destination match the source. If a file is accidentally deleted from the primary Maildir (operator error, bug, filesystem corruption), the next rclone run propagates that deletion to B2 and R2. Object versioning must be enabled on both buckets so previous versions are retained even after a sync delete. This mirrors the protection rsnapshot provides for the local copy.

**Why not restic or duplicati?**
- restic is excellent but pack-based; Maildir's one-file-per-message structure is better served by a mirror-style sync that preserves individual files for the deletion worker to verify
- rclone crypt keeps file granularity intact, so backup verification is `rclone check` rather than a full restore test

---

## 2. Deletion Worker — State Tracking

**Decision: SQLite manifest database on the primary server**

The deletion worker needs to know, for every message, whether all backup copies exist. A lightweight embedded database is sufficient given email volume and single-server deployment.

### Schema (conceptual)

```
messages
  message_id        — Message-ID header value (PK, globally unique per RFC 5322)
  content_hash      — SHA-256 of (From + To + Date + Subject + body), for integrity checks
  first_seen_at     — timestamp of first ingest across any account
  hold              — boolean, blocks deletion from all provider copies

provider_copies
  id                — surrogate PK
  message_id        — FK → messages.message_id
  account           — provider account identifier (e.g. "gmail-work")
  provider_uid      — IMAP UID from the provider
  maildir_path      — relative path within this account's Maildir namespace
  ingested_at       — timestamp this copy was ingested
  deletion_status   — enum: ELIGIBLE | PENDING | DELETED | HOLD
  failure_count     — consecutive IMAP deletion failure count (reset on success)
  deleted_at        — timestamp of confirmed provider deletion (null until DELETED)

backup_verifications
  provider_copy_id  — FK → provider_copies.id         ┐ composite
  destination       — enum: LOCAL | B2 | R2            ┘ PRIMARY KEY
  confirmed_at      — timestamp of last successful rclone check
  status            — enum: PENDING | CONFIRMED | MISSING

account_policy
  account           — PK, matches provider_copies.account
  retention_days    — override global default (null = use global)
  verification_window_days — override global default (null = use global)
  hold_all          — boolean, blocks all deletions for this account
```

### Why three tables

The original two-table schema collapsed `messages` and `provider_copies` into one row, making it impossible to represent the same message appearing in multiple provider accounts. Separating them allows each provider copy to be tracked and deleted independently.

### Message-ID as primary key

`Message-ID` is the semantically correct identity for a message across delivery contexts. The raw RFC 5322 hash cannot serve this role — the same message delivered to two accounts will have different `Received:` headers, producing different hashes.

Edge cases:
- **Missing Message-ID** — synthesize one from `content_hash`
- **Message-ID collision** (two different messages, same header value) — detect via `content_hash` mismatch; store both with a disambiguating suffix on the PK

### Maildir storage: per-account namespaces

Each account gets its own Maildir subdirectory (e.g. `Maildir/gmail-work/`, `Maildir/outlook-personal/`). No file-level deduplication across accounts. Cross-account messages that appear as duplicates are rare in practice (different `Received:` headers → different raw bytes), and per-account namespaces keep Dovecot configuration simple and each account's mail self-contained.

### Verification flow

1. After each rclone sync run, run `rclone check` against each destination
2. Update `backup_verifications.confirmed_at` and `status` for all present files
3. The deletion worker queries: `provider_copies` where `deletion_status = ELIGIBLE` AND `ingested_at < now − retention_days`, where ALL three `backup_verifications` for that copy have `status = CONFIRMED` AND `confirmed_at > now − verification_window_days`

Each provider copy is evaluated **independently** — the deletion eligibility of one account's copy of a message does not depend on the backup status of another account's copy of the same message.

### Worker algorithm

The deletion worker runs as a long-running Python daemon on a configurable cycle (default: every 6 hours).

**One cycle:**

```
0. On startup only: reset PENDING rows older than one cycle duration → ELIGIBLE
   (handles crash-recovery; runs before any HALT checks)

1. PRAGMA integrity_check
   → if not "ok": HALT immediately (emit ManifestIntegrityFailure alert)

2. Query: provider_copies WHERE deletion_status = ELIGIBLE
                              AND ingested_at < NOW - retention_window

3. Group candidates by account. For each account:
   a. Open one IMAP session (connect + authenticate)
   b. For each candidate message in this account:
      - Check: all 3 backup_verifications WHERE status = CONFIRMED
                                             AND confirmed_at > NOW - verification_window
      - If any backup NOT confirmed:
          skip (no state change); increment warning metric
      - If all confirmed:
          SET deletion_status = PENDING   ← excludes from future cycles until resolved
          Issue IMAP UID STORE +FLAGS \Deleted + EXPUNGE
          On success: SET deletion_status = DELETED, record deleted_at
                      increment worker_deletions_total{status="success"}
          On failure: SET deletion_status = ELIGIBLE, increment failure_count
                      increment worker_deletions_total{status="failed"}
                      if failure_count > 5: emit DeletionFailure alert
   c. Close IMAP session

4. Push metrics to Pushgateway

5. POST heartbeat to healthchecks.io

6. Sleep until next cycle
```

**HALT conditions:**

| Condition | Scope | Alert |
|---|---|---|
| `PRAGMA integrity_check` non-"ok" | All accounts — full stop | `ManifestIntegrityFailure` |
| Manifest DB inaccessible (lock failure, file missing) | All accounts — full stop | `WorkerHalted` |
| All three backup destinations MISSING for 3+ consecutive cycles | All accounts — full stop | `WorkerHalted` + `BackupSyncStale` |
| 5 consecutive IMAP auth failures for an account | That account only — others continue | `AuthFailure` |

A single failed deletion does not halt the worker. The `DeletionFailure` alert fires when `failure_count > 5` for a given message.

**PENDING prevents double-deletion:** The query in step 2 selects only `deletion_status = ELIGIBLE`. A row set to PENDING is invisible to future cycles until explicitly reset to ELIGIBLE (on failure) or DELETED (on success). Step 0 handles crash-recovery by resetting stale PENDING rows before the HALT checks run.

### Manifest DB resilience

The `manifest-db` volume must be included in the rclone sync scope alongside `maildir`. Loss of the manifest without a backup means the deletion worker cannot function and backup verification state must be rebuilt by scanning the Maildir and re-running `rclone check` against all three destinations — possible but slow.

Two additional requirements at the implementation layer:
- **WAL mode** (`PRAGMA journal_mode=WAL`) — enables concurrent reads while a write is in progress; the manifest is accessed by both mbsync (writer) and the worker (reader/writer) and WAL mode is the correct default for this pattern
- **Periodic integrity check** (`PRAGMA integrity_check`) — should run on a schedule (e.g. daily) and alert if it returns anything other than `ok`

---

## 3. Retention Policy

**Decision: time-based with a per-account override table**

### Global default

A message becomes eligible for provider-side deletion when:
- It has been in the primary archive for at least **N days** (default: 90)
- All three backup copies have been confirmed within the last **M days** (default: 7)

### Per-account overrides

Some accounts (e.g. a work inbox) may need longer retention or indefinite hold. An `account_policy` table overrides the global defaults per account.

### What "deletion" means

- The message is **not** deleted from the primary Maildir or any backup
- Only the copy on the provider's server (Gmail, Outlook, etc.) is deleted
- This frees provider storage and reduces ongoing OAuth/credential exposure

### Retention hold

A message with a `hold = true` flag is never eligible for provider deletion regardless of policy. Useful for legal hold or manually flagged threads.

---

## 4. Error Handling for Partial Backup Failures

**Decision: conservative gating — all copies must be CONFIRMED, no deletions on degraded state**

### Failure modes and responses

| Failure | Behavior |
|---|---|
| One backup destination unreachable | Deletion worker pauses all deletions; alert operator |
| rclone check finds missing file | Set `status = MISSING`; re-sync; do not delete from provider |
| Primary archive disk full | Ingest pauses; backups continue; alert operator |
| All backups confirmed but primary Maildir file missing | Treat as critical integrity error; halt deletion worker; alert |
| Provider IMAP deletion fails | Mark deletion attempt; retry with backoff; do not double-count |

### Alerting

The deletion worker emits structured log lines. A separate alerting process (or cron job) reads logs and sends notifications (email, webhook, etc.) when any failure state is entered.

### No partial deletes

The worker processes messages in atomic batches per account. If any message in a batch fails the backup check, the entire batch is skipped — no partial deletion of an account's messages in one run.

### Ingest overlap prevention

Ofelia triggers mbsync on a fixed schedule (e.g. every 15 minutes). If a sync run takes longer than the interval — large mailbox, slow provider, throttling — Ofelia will spawn a second mbsync container while the first is still writing to the Maildir. Two concurrent mbsync instances writing to the same account's Maildir will corrupt the UID cache and can produce duplicate messages. All ingest jobs must be configured with `no-overlap: true` in Ofelia, which skips a scheduled run if the previous instance has not exited.

---

## 5. Provider Authentication

**Decision: OAuth2 for Gmail and Outlook; app passwords for Yahoo and generic IMAP**

### Gmail

- OAuth2 via Google's device authorization flow (no browser needed on server)
- Scopes: `https://mail.google.com/` (full IMAP access)
- Refresh tokens stored in the server's secret store
- mbsync configured to use OAuth2 XOAUTH2 SASL mechanism via external password command
- **Sync only `[Gmail]/All Mail`** — every Gmail message has a unique UID there; syncing additional label-folders (Inbox, Sent, etc.) would produce within-account duplicates

### Outlook / Microsoft 365

- OAuth2 via Microsoft identity platform (MSAL device flow)
- Scopes: `https://outlook.office.com/IMAP.AccessAsUser.All`
- Refresh tokens stored in the server's secret store

### Yahoo and generic IMAP

- App-specific passwords (Yahoo disabled standard password IMAP)
- Credentials stored in the server's secret store, referenced by ingest config

### Secret store

All credentials (OAuth2 refresh tokens, app passwords, rclone crypt passphrases) are stored in a single secret store on the VPS. In a minimal self-hosted deployment, this is an `age`-encrypted file unlocked at startup. In a more hardened deployment, HashiCorp Vault or a similar tool is used.

Credentials are **never** in config files committed to version control.

---

## 6. Resilience & Reliability

### External heartbeat (dead man's switch)

Every alert in the observability stack depends on Prometheus evaluating rules. Prometheus runs on the same VPS as everything else. If the VPS goes down, Prometheus goes down with it and no alerts fire — the operator receives silence instead of a page.

The system must push a periodic heartbeat to an external service independent of the VPS (e.g. healthchecks.io). If the heartbeat stops arriving, the external service sends the notification. The heartbeat should be emitted by the worker after each successful poll cycle and by Ofelia after each mbsync run. This is the only mechanism that can detect a total VPS outage.

### VPS hardening baseline

Required before deployment:
- SSH password authentication disabled; key-only login
- SSH on non-standard port or protected by fail2ban
- Unattended security upgrades enabled on the VPS OS
- Deploy SSH user is in the `docker` group only — no sudo, no system access

### At-rest encryption

LUKS full-disk encryption is impractical for unattended cloud VPS deployments — it requires interactive passphrase entry at every boot. The chosen approach is **gocryptfs + Clevis/Tang**, documented in §7. The Maildir and manifest DB are encrypted at the directory level; the passphrase is derived automatically at startup from the Tang server over Tailscale and never stored on the primary VPS disk.

### Dovecot network exposure

Dovecot is a long-running daemon with a network-accessible IMAP port. The following must be defined before deployment:

- **TLS required** — Dovecot must terminate IMAPS (port 993) with a valid certificate. Plaintext IMAP (port 143) must not be accessible outside localhost.
- **Access scope** — Dovecot is accessible only to Tailscale network peers. **Tailscale ACLs must explicitly restrict which nodes can reach VPS port 993** — not all tailnet peers, only designated mail client devices and the local backup server.
- **Certificate source** — Tailscale provides per-machine TLS certificates via its ACME integration; this is the natural fit for a Tailscale-scoped deployment.
- **Dovecot authentication** — method to be defined at implementation time (passwd-file is the minimal self-contained option).

### Container restart policies

All long-running containers must be configured with `restart: unless-stopped` in Docker Compose. Without explicit restart policies, a crashed worker, Dovecot, or Alertmanager container stays down silently. The `WorkerHalted` alert catches a running-but-halted worker, but it cannot catch a crashed and not-restarted worker process.

### RPO and RTO

**Recovery Point Objective:** Near-zero for practical purposes. The deletion worker will not remove a message from a provider until 90 days of confirmed archiving has passed. If the VPS is down for days or weeks, email remains on the provider and will be ingested when the system resumes. The only scenario with real data loss risk is if ingest is silently broken for over 90 days and the provider deletes messages before they are ever archived — this is why the external heartbeat is critical.

**Recovery Time Objective:** Not formally defined. The recovery procedure is documented in `docs/recovery.md`. Rough estimate: 1–4 hours depending on Maildir volume (dominated by rclone decrypt + restore time from B2 or R2).

---

## Component Map

```
┌─────────────────────────────────────────────────┐
│                   VPS / Server                  │
│                                                 │
│  ┌──────────┐    ┌─────────────────────────┐   │
│  │  Ingest  │───▶│  Central Maildir Archive │   │
│  │  daemon  │    │  (gocryptfs encrypted)   │   │
│  └──────────┘    └────────────┬────────────┘   │
│       │                       │                 │
│  ┌────▼──────┐         ┌──────▼──────┐         │
│  │  Manifest │         │   Dovecot   │         │
│  │   DB      │         │ (IMAPS/993) │         │
│  │ (SQLite)  │         │ Tailscale   │         │
│  └────┬──────┘         │ peers only  │         │
│       │                └─────────────┘         │
│  ┌────▼──────────┐                              │
│  │Deletion Worker│──────────────────────────┐  │
│  └───────────────┘                           │  │
│                                              ▼  │
│  ┌─────────────────────────────────────────────┐│
│  │  rclone sync (crypt overlay)                ││
│  │  → Tailscale VPN → Local rsnapshot          ││
│  │  → Backblaze B2 (encrypted, versioned)      ││
│  │  → Cloudflare R2 (encrypted, versioned)     ││
│  └─────────────────────────────────────────────┘│
│                                                 │
│  Heartbeat ──────────────────▶ healthchecks.io │
└─────────────────────────────────────────────────┘
```

---

## 7. Data At Rest — gocryptfs + Clevis/Tang

**Decision: gocryptfs directories unlocked at startup via Clevis + Tang on a secondary VPS**

LUKS full-disk encryption requires interactive passphrase entry at boot, making it impractical for an unattended cloud VPS. gocryptfs provides directory-level AES-256-GCM encryption with per-file IVs, requiring no kernel block device or reboot sequence. The unlock passphrase is never stored on the primary VPS disk — it is derived at startup via the Tang key-agreement protocol.

### What is encrypted

| Directory | Encrypted | Key source |
|---|---|---|
| Maildir (all per-account subdirs) | Yes — gocryptfs | Tang server (secondary VPS) |
| Manifest DB (SQLite) | Yes — gocryptfs | Tang server (secondary VPS) |
| OS, logs, container layers | No | — |
| rclone config, Docker secrets | No (tmpfs Docker secrets) | — |

### Tang and the Clevis protocol

Tang is a stateless key-agreement server implementing JOSE ECDH. The client (Clevis on the primary VPS) and Tang server perform an elliptic curve Diffie-Hellman exchange. The passphrase is **derived** from the exchange — it is never transmitted in plaintext, never stored by Tang, and never persisted on the primary VPS disk. Both sides of the exchange are required to reproduce the passphrase.

```
Primary VPS startup:
  Clevis → Tang (secondary VPS, Tailscale) → ECDH exchange
  Clevis derives passphrase locally from exchange result
  gocryptfs mounts /maildir and /manifest-db
  All containers start after mount succeeds

Primary VPS disk imaged offline:
  Tang unreachable → Clevis cannot complete exchange
  gocryptfs passphrase cannot be derived
  Maildir and manifest DB remain ciphertext
```

### Secondary VPS as Tang server

Tang is extremely lightweight — it is a small daemon serving JOSE key-agreement responses. It requires no persistent storage of keys or client state. A minimal VPS instance (512 MB RAM, 1 vCPU) is sufficient.

The Tang server is added to the Tailscale tailnet. The primary VPS contacts it only at startup (and during periodic re-binding). The Tang server holds no email data.

**Advantages over Tang on the local backup server:**
- Always-on cloud availability (local server may be off or hibernated)
- Separate failure domain from the primary VPS
- No dependency on local server for VPS startup

### gocryptfs properties

- AES-256-GCM with authenticated encryption
- Per-file random IVs — compromise of one file's ciphertext does not expose key material for any other file (the at-rest analog of forward secrecy within a key epoch)
- File names optionally encrypted (enable for Maildir — leaks no folder/account structure at the filesystem level)
- Reverse mode not used (standard forward encryption)

### What this protects against

| Threat | Protected? |
|---|---|
| VPS disk image / snapshot leaked by cloud provider | Yes — ciphertext without Tang |
| Physical disk extraction | Yes — Tang unreachable offline |
| Cloud provider reads disk | Yes |
| Live VPS compromise (attacker has shell) | No — gocryptfs is mounted and decrypted in memory |
| Memory dump of running processes | No — same limitation as all at-rest encryption |

At-rest encryption is specifically a disk-image / physical access threat mitigator. A live system compromise still has access to the mounted plaintext. VPS hardening (§6) remains the primary defense against live compromise.

### Local rsnapshot backup

The local server receives a plaintext rsync of the Maildir over Tailscale (gocryptfs is a filesystem mount — rclone sees the decrypted files). The local server should apply its own at-rest encryption:
- **Preferred:** LUKS on the local server's backup volume (physical access is possible for manual unlock)
- **Alternative:** gocryptfs on the rsnapshot target, passphrase stored locally (local server is trusted)

### Startup sequence

gocryptfs is a FUSE filesystem. FUSE mounts are per-mount-namespace: a mount inside Docker container A is invisible to container B. The correct approach is a **host-level mount before Docker starts** — no privileged containers needed, all containers bind-mount the already-decrypted host paths.

```
VPS boot
  └─ gocryptfs-mount.service          (systemd, Before=docker.service,
  │                                    After=network-online.target)
  │    ├─ clevis decrypt < /etc/mailarchiver/tang-binding.jwe
  │    │    → derives passphrase via ECDH with Tang server over Tailscale
  │    ├─ gocryptfs -passfile /dev/stdin \
  │    │    /var/lib/mailarchiver/maildir-cipher \
  │    │    /var/lib/mailarchiver/maildir
  │    ├─ gocryptfs -passfile /dev/stdin \
  │    │    /var/lib/mailarchiver/manifest-db-cipher \
  │    │    /var/lib/mailarchiver/manifest-db
  │    └─ (passphrase never written to disk; derived in memory only)
  │
  └─ docker.service  (starts only after gocryptfs-mount.service succeeds)
       └─ docker compose up
            (mbsync, dovecot, worker, rclone bind-mount
             /var/lib/mailarchiver/maildir and /manifest-db as ordinary volumes)
```

The systemd ordering guarantees that if `gocryptfs-mount.service` fails, Docker does not start and containers never see an unmounted path. No healthcheck polling required.

**If Tang is unreachable at startup:** `clevis decrypt` fails, the systemd unit exits non-zero, Docker does not start. Resolution options:
1. Restore Tang connectivity (normal case — secondary VPS is down or Tailscale is disrupted) and reboot
2. Manual fallback — see the fallback procedure below and `docs/operations.md §7`

### Fallback / key rotation

- **Fallback if Tang unreachable:** A second Clevis binding using an age-encrypted passphrase (escrowed in password manager) allows manual mount. See `docs/operations.md §7` for the exact procedure.
- **Key rotation:** To rotate, generate a new Tang binding on the gocryptfs volume, revoke the old one. No re-encryption of data needed — gocryptfs master key stays the same; only the Tang binding changes. See `docs/operations.md §6`.

---

## Open Decisions (deferred)

- **Dovecot authentication method** — passwd-file vs PAM vs other; to be decided at implementation time.

## Resolved Decisions

- **Ingest tool** — mbsync (isync). See `docs/tech-stack.md`.
- **Deduplication** — three-table manifest schema; `Message-ID` as deduplication key; per-account Maildir namespaces; independent deletion per provider copy. See §2.
- **Observability stack** — Prometheus + Loki + Grafana + Alertmanager. See `docs/observability.md`.
- **Resilience gaps** — external heartbeat, B2/R2 object versioning, manifest DB backup + WAL, ingest overlap prevention, passphrase escrow, Dovecot network scope, restart policies, RPO/RTO. See §6 and `docs/recovery.md`.
- **At-rest encryption** — gocryptfs on Maildir + manifest DB, passphrase derived via Clevis + Tang on secondary VPS over Tailscale. See §7.
