# mailarchiver — Threat Model

## Assets

Ranked by impact of loss or compromise:

| Asset | Impact of compromise |
|---|---|
| **Email content** (Maildir, backups) | Irreversible data loss or exposure of all personal correspondence |
| **rclone crypt passphrase** | B2 and R2 backups permanently unrecoverable |
| **OAuth2 refresh tokens** | Attacker gains full read/write/delete access to provider accounts |
| **Deletion worker integrity** | Worker subverted into deleting provider copies before backup confirmed → permanent loss |
| **Manifest DB** | Loss of deletion tracking state; recovery requires full Maildir rescan |
| **System availability** | Ingest stops; email accumulates on provider indefinitely |

---

## Threat Actors

| Actor | Capability | Primary motivation |
|---|---|---|
| Remote attacker | Network-level access, vulnerability exploitation | Email exfiltration, credential theft |
| Compromised dependency | Malicious code in Docker image or Python package | Exfiltration, persistence, pivoting |
| Compromised CI/CD | Control over deployment pipeline | VPS access, malicious image deployment |
| Cloud storage provider | Read access to stored objects | Surveillance (mitigated by rclone crypt) |
| Physical attacker | Access to local backup server hardware | Unencrypted Maildir exposure |
| Misconfiguration / self | Accidental operator action | Data loss, credential exposure |

---

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│ Internet                                                        │
│  Gmail, Outlook, Yahoo  ──IMAP/TLS──▶                          │
│  Backblaze B2, CF R2    ◀──S3/TLS──▶   ┌────────────────────┐ │
│  GitHub Actions         ──SSH──────▶   │       VPS          │ │
│  healthchecks.io        ◀──HTTPS───    │                    │ │
│  SMTP relay             ◀──SMTP/TLS─   │  ┌──────────────┐  │ │
│  Slack/Discord webhook  ◀──HTTPS───    │  │ app-net      │  │ │
└─────────────────────────────────────────  │  (mbsync,     │  │ │
                                         │  │  worker,      │  │ │
┌────────────────────┐                   │  │  dovecot,     │  │ │
│ Tailscale network  │                   │  │  rclone,      │  │ │
│                    │                   │  │  tailscale,   │  │ │
│  Mail clients      ├──IMAPS/993──▶     │  │  ofelia)      │  │ │
│  Local server      ├──rsync──────▶     │  └──────────────┘  │ │
│  (rsnapshot)       │                   │  ┌──────────────┐  │ │
└────────────────────┘                   │  │ obs-net      │  │ │
                                         │  │ (prometheus, │  │ │
                                         │  │  loki,       │  │ │
                                         │  │  grafana,    │  │ │
                                         │  │  alertmgr,   │  │ │
                                         │  │  promtail,   │  │ │
                                         │  │  pushgw,     │  │ │
                                         │  │  node-exp)   │  │ │
                                         │  └──────────────┘  │ │
                                         └────────────────────┘ │
```

---

## Threats and Mitigations

### T1 — VPS Compromise (SSH / Vulnerability Exploitation)

**Impact:** Full access to unencrypted Maildir, all Docker secrets, deletion worker manipulation.

**Vectors:** SSH brute force; exploitation of a running service; compromised deploy key.

| Mitigation | Severity | Status |
|---|---|---|
| Disable SSH password authentication; key-only | Critical | Not yet defined |
| SSH on non-standard port or port-knock; fail2ban | High | Not yet defined |
| Deploy SSH user is least-privilege (docker group only, no sudo) | High | Defined in cicd.md |
| Minimize public attack surface: only Dovecot exposed, and only to Tailscale | High | Defined in architecture.md |
| Unattended security upgrades on VPS OS | High | Not yet defined |
| Regular rotation of deploy SSH key (`VPS_SSH_KEY`) | Medium | Not yet defined |

---

### T2 — Maildir at Rest

**Impact:** All email readable if VPS disk is imaged (cloud provider breach, physical disk access, snapshot leak).

**Decision:** gocryptfs on the Maildir and manifest DB directories, passphrase derived at startup via Clevis + Tang on a secondary VPS over Tailscale. See `architecture.md §7`.

| Mitigation | Severity | Status |
|---|---|---|
| gocryptfs AES-256-GCM on `/maildir` and `/manifest-db` — ciphertext at rest, decrypted only while VPS is running | High | **Defined in architecture.md §7** |
| Tang server on secondary VPS — passphrase derived via JOSE ECDH; never stored on primary disk; never transmitted in plaintext | High | **Defined in architecture.md §7** |
| Per-file random IVs in gocryptfs — file-level isolation; compromise of one file's ciphertext does not expose key material for others | Medium | Inherent to gocryptfs |
| File name encryption enabled — leaks no folder/account structure at filesystem level | Medium | **Defined in architecture.md §7** |
| Fallback binding: age-encrypted passphrase escrowed in password manager (manual mount if Tang unreachable) | Medium | **Defined in architecture.md §7** |
| Local rsnapshot copy: LUKS on local server backup volume (physical unlock possible) | Medium | See T9 |

**Residual risk:** A live VPS compromise (attacker has shell) still has access to the mounted plaintext. gocryptfs is a disk-image threat mitigator, not a live-compromise mitigator. VPS hardening (T1) remains the primary defense against live access.

**Note:** B2 and R2 copies are independently encrypted by rclone crypt. The local rsnapshot copy is addressed separately in T9.

---

### T3 — OAuth2 Token Theft

**Impact:** Attacker reads, deletes, or exfiltrates email directly from provider accounts.

**Vectors:** VPS compromise → Docker secrets; accidental token logging; mbsync debug output.

| Mitigation | Severity | Status |
|---|---|---|
| OAuth2 scopes minimized: request only what is needed (see T4) | High | Needs review — see T4 |
| Tokens stored as Docker secrets (tmpfs, never written to disk) | High | Defined in architecture.md |
| Never log OAuth2 tokens or Authorization headers | High | Implementation requirement |
| Documented token revocation procedure (per provider) | Medium | Not yet defined |
| Periodic OAuth2 token rotation (re-authorize on a schedule) | Medium | Not yet defined |

---

### T4 — OAuth2 Scope Excess

**Impact:** If tokens are stolen, attacker has broader account access than needed.

**Current design:** Gmail scope documented as `https://mail.google.com/` (full IMAP access, including delete). This is required for the deletion worker to delete provider copies.

| Mitigation | Severity | Status |
|---|---|---|
| For **ingest-only** accounts (where deletion from provider is disabled or deferred): request read-only IMAP scope | Medium | Not yet defined |
| For **deletion-enabled** accounts: full IMAP scope is unavoidable; compensate with token rotation and VPS hardening | — | Accept |
| Outlook scope: `IMAP.AccessAsUser.All` is correct minimal scope for both read and delete | Low | Defined in architecture.md |

---

### T5 — Supply Chain: Docker Image Compromise

**Impact:** Malicious code running on VPS with access to shared volumes (Maildir, manifest DB) and Docker internal network.

**Vectors:** Compromised off-the-shelf image on Docker Hub; mutable tag (`latest`) hijacked; typosquatting.

| Mitigation | Severity | Status |
|---|---|---|
| **Pin all off-the-shelf images by digest** (`image@sha256:...`), not by tag | Critical | **Not yet defined — gap** |
| **Trivy scan off-the-shelf images** in CI alongside custom images | High | Currently CI only scans custom images — extend |
| Use Renovate/Dependabot to track digest updates (opens PR when upstream digest changes) | High | Defined in cicd.md — extend to digest pinning |
| Prefer images from official namespaces (e.g. `prom/prometheus`, `grafana/grafana`) over community variants | Medium | Implementation guidance |
| Docker Content Trust / Cosign signature verification for custom images | Medium | Not yet defined |

---

### T6 — Supply Chain: GitHub Actions Compromise

**Impact:** CI workflow runs malicious code with `VPS_SSH_KEY` in scope → persistent VPS access.

**Vectors:** Third-party action compromised (`actions/checkout@v4` tag moved); maintainer account compromised; malicious PR modifying workflow files.

| Mitigation | Severity | Status |
|---|---|---|
| **Pin all GitHub Actions to commit SHA**, not tag (`actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` not `@v4`) | Critical | **Not yet defined — gap** |
| Protect `main` branch: require PR + CI pass; no direct push | High | Not yet defined |
| Require PR review before merge to main | High | Not yet defined |
| Workflow permissions: set `permissions: read-all` at top level; grant write only where needed | High | Not yet defined |
| `VPS_SSH_KEY` scoped to least-privilege deploy user (docker only, no sudo) | High | Defined in cicd.md |
| Audit third-party actions; prefer official GitHub actions or well-audited equivalents | Medium | Not yet defined |

---

### T7 — Tailscale ACL / Rogue Peer

**Impact:** Any Tailscale peer can reach Dovecot IMAP (read all archived email) and the local backup server.

**Vectors:** Tailscale account compromise; rogue device added to tailnet; Tailscale service compromise.

| Mitigation | Severity | Status |
|---|---|---|
| **Define Tailscale ACLs**: only specific nodes (mail clients, local server) can reach VPS:993 | High | **Not yet defined — gap** |
| Enable Tailscale device approval: new devices require explicit admin approval before joining tailnet | High | Not yet defined |
| Enable Tailscale MFA on the tailnet admin account | High | Not yet defined |
| Dovecot authentication provides a second factor for IMAP access even for authorized Tailscale peers | Medium | Method TBD — see architecture.md |
| Monitor Tailscale admin console for unexpected nodes | Low | Not yet defined |

---

### T8 — Cloud Storage Provider Breach (B2 / R2)

**Impact:** Provider reads stored backup data.

**Current design:** rclone crypt encrypts all data before it leaves the VPS. File names and content are opaque to the provider.

| Mitigation | Severity | Status |
|---|---|---|
| rclone crypt AES-256-CTR already in design | — | Defined in architecture.md |
| **B2/R2 API keys scoped to the specific backup bucket only** (not account-level) — limits blast radius if API key is leaked | High | **Not yet defined — gap** |
| Enable bucket-level access policies: block public access, no pre-signed URL generation | High | Not yet defined |
| Enable object versioning (already required for accidental deletion protection — see resilience doc) | High | Defined in architecture.md |
| Consider B2/R2 Object Lock (WORM) for additional tamper protection | Low | Not yet defined |

---

### T9 — Local Backup Exposure

**Impact:** Physical or network access to local server exposes unencrypted Maildir.

**Current design:** B2 and R2 are encrypted. The local rsnapshot copy is **not encrypted** (it receives the plaintext Maildir over the Tailscale VPN).

| Mitigation | Severity | Status |
|---|---|---|
| **Full-disk encryption on the local backup server** (LUKS or equivalent) | High | **Not yet defined — documented gap** |
| rsnapshot directory: readable only by the rsnapshot user (filesystem permissions) | Medium | Not yet defined |
| Access to local server itself only via Tailscale (same ACL controls as VPS access) | Medium | Implicitly handled by Tailscale network — define explicitly |
| Accept residual risk: local backup is unencrypted at rest by design | — | Document as accepted risk if disk encryption is not implemented |

---

### T10 — Docker Network (Flat Internal Network)

**Impact:** A compromised observability container (Prometheus, Grafana, Loki, etc.) has network access to the worker, Pushgateway, and can scrape Maildir-related metrics. A compromised container could push false metrics to suppress alerts.

**Current design:** Single flat Docker Compose network — all containers can reach each other.

| Mitigation | Severity | Status |
|---|---|---|
| **Define two Docker networks**: `app-net` (application containers) and `obs-net` (observability containers) | High | **Not yet defined — gap** |
| Worker attached to both networks (exposes `/metrics` to `obs-net`; accesses Maildir on `app-net`) | High | Design required |
| Pushgateway on `app-net` only (mbsync and rclone push to it); Prometheus scrapes it from `obs-net` via a bridge | Medium | Design required |
| Promtail requires Docker socket access (host bind mount) — this is unavoidable; review what other containers mount | Medium | Not yet defined |

**Proposed network layout:**

```
app-net:   mbsync, dovecot, worker, rclone, tailscale, ofelia, pushgateway
obs-net:   prometheus, loki, grafana, alertmanager, promtail, node-exporter
both:      worker (metrics bridge)
```

---

### T11 — Pushgateway Metric Injection

**Impact:** A compromised container pushes false metrics (e.g., fake `CONFIRMED` backup status) to suppress alerts or mask failures.

| Mitigation | Severity | Status |
|---|---|---|
| Restrict Pushgateway to `app-net` — only mbsync and rclone can reach it | Medium | Requires T10 network segmentation |
| Prometheus alerting rules should cross-reference manifest DB state (worker metrics) independently of Pushgateway values | Medium | Implementation requirement for worker |
| Consider Pushgateway `--web.enable-admin-api=false` to prevent arbitrary job deletion | Low | Not yet defined |

---

### T12 — Grafana Unauthorized Access

**Impact:** Email metadata visible (account names, message counts, backup status, deletion history) — operational intelligence about personal correspondence patterns.

| Mitigation | Severity | Status |
|---|---|---|
| Grafana: enable authentication (username/password minimum; Tailscale SSO if available) | High | **Not yet defined** |
| Grafana: disable anonymous access (`allow_sign_up = false`, `allow_embedding = false`) | High | Not yet defined |
| Expose Grafana only on Tailscale network (not on public VPS IP) | High | Not yet defined — consistent with Dovecot scope |

---

### T13 — Secret Rotation Absence

**Impact:** Compromised credentials remain valid indefinitely.

| Credential | Rotation cadence | Rotation procedure |
|---|---|---|
| Deploy SSH key (`VPS_SSH_KEY`) | Every 90 days or on personnel change | Rotate in GitHub Secrets + `authorized_keys` on VPS |
| OAuth2 refresh tokens | As needed (Google: 6-month inactivity expiry; Microsoft: 90 days without use) | Re-run device authorization flow; update Docker secret |
| App passwords (Yahoo, generic) | Annually or on suspected compromise | Generate new password in provider account; update Docker secret |
| rclone crypt passphrase | Do not rotate (rotation requires re-encrypting all B2/R2 data) | If compromised: re-encrypt all backups with new passphrase |
| SMTP relay password | Annually | Update Docker secret + verify alert delivery |
| Slack/Discord webhook URL | On suspected compromise | Regenerate in provider; update Docker secret |

---

### T14 — Container Resource Exhaustion

**Impact:** A runaway container (e.g., Loki ingesting excessive log volume) starves the worker or Dovecot.

| Mitigation | Severity | Status |
|---|---|---|
| Define Docker resource limits (`mem_limit`, `cpus`) per container in Compose | Medium | **Not yet defined** |
| Loki and Prometheus retention limits prevent unbounded disk growth | Medium | Partially defined (retention periods deferred) |
| `MaildirDiskPressure` alert already covers disk exhaustion | — | Defined in observability.md |

---

## Summary: Gaps Requiring New Decisions

| Gap | Where to address | Severity |
|---|---|---|
| LUKS encryption for VPS data volume | architecture.md §6 + ops runbook | High |
| Pin all Docker images by digest | tech-stack.md + cicd.md | Critical |
| Extend Trivy scan to off-the-shelf images | cicd.md | High |
| Pin GitHub Actions by commit SHA | cicd.md | Critical |
| Protect `main` branch in GitHub | cicd.md | High |
| Tailscale ACLs (restrict Dovecot access to specific nodes) | architecture.md §6 | High |
| B2/R2 API keys scoped to bucket, not account | architecture.md §1 | High |
| Local backup disk encryption | recovery.md + architecture.md | High |
| Docker network segmentation (`app-net` / `obs-net`) | tech-stack.md | High |
| Grafana authentication | observability.md | High |
| Secret rotation cadences | architecture.md §5 | Medium |
| Container resource limits | tech-stack.md | Medium |

---

## Accepted Risks

| Risk | Rationale |
|---|---|
| Maildir unencrypted at rest (if LUKS not implemented) | Compensating controls: minimal attack surface, Tailscale-only exposure, SSH hardening. VPS security is the primary control. |
| Local backup unencrypted at rest (if local disk encryption not implemented) | Tailscale-scoped access, rsnapshot permissions provide soft mitigation. Physical security of the local server is assumed. |
| Tailscale vendor dependency | If Tailscale is compromised at the vendor level, the VPN trust model fails. Mitigated by Dovecot authentication as a second layer. |
| rclone crypt passphrase is a single key for all offsite data | Passphrase escrow requirement addresses loss; compromise requires immediate re-encryption of all backups. |
