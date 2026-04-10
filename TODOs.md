# mailarchiver — Documentation Gaps, Undefined Items, and Ambiguities

This file tracks every item across the project docs that is undefined, ambiguous, or a known gap. It is a living document — update the Status column as items are resolved during implementation.

**Status values:** `Open` | `In Progress` | `Resolved`

Cross-references use `docs/X.md §N` format. Threat model references are T1–T18 per `docs/threat-model.md`.

---

## Critical — Must Resolve Before Any Deployment

These items are security gaps or procedural errors that could cause data loss or exposure if ignored.

| ID | Item | Source | Threat | Status | Resolved in |
|---|---|---|---|---|---|
| C1 | Recovery runbook missing gocryptfs init + mount step before `rclone copy` — restoring to an unmounted path writes decrypted email to unencrypted disk | `docs/recovery.md §Step 3` | T18 | Resolved | `docs/recovery.md §Step 3` |
| C2 | VPS host firewall not defined — no ufw/iptables default-deny rule; observability services could be accidentally port-mapped to the public interface | No doc covers this | T17 | Resolved | `docs/architecture.md §VPS host firewall` |
| C3 | Docker socket proxy for Promtail not implemented — Promtail's `/var/run/docker.sock` bind mount gives full Docker API access, bypassing app-net/obs-net segmentation | `docs/tech-stack.md §Docker network segmentation` | T16 | Resolved | `docs/tech-stack.md §Docker socket proxy` |
| C4 | Tailscale ACLs not defined — Dovecot port 993 and Tang port 7500 are accessible to any tailnet peer; must restrict to specific nodes only | `docs/architecture.md §6` | T7, T15 | Resolved | `docs/architecture.md §Tailscale ACL matrix` |

---

## High — Security-Relevant, Resolve Before the Affected Iteration

| ID | Item | Source | Threat | Status | Resolved in |
|---|---|---|---|---|---|
| H1 | Tang server VPS hardening baseline not defined — same requirements as primary VPS (SSH key-only, fail2ban, unattended-upgrades) but no doc covers Tang | `docs/threat-model.md §T15` | T15 | Resolved | `docs/architecture.md §Tang VPS hardening baseline` |
| H2 | Local backup disk encryption decision deferred — no chosen at-rest encryption model for the rsnapshot target | `docs/architecture.md §Local rsnapshot backup` | T9 | Resolved | `docs/architecture.md §Local rsnapshot backup` |
| H3 | Dovecot authentication method explicitly deferred (passwd-file vs PAM) — must be decided before Iteration 2 Track A begins | `docs/architecture.md §Open Decisions` | T1 | Open | |
| H4 | SSH hardening specifics not defined — non-standard port value and fail2ban config thresholds are unspecified; "Not yet defined" in threat model | `docs/threat-model.md §T1` | T1 | Open | |
| H5 | Tailscale device approval and tailnet admin MFA — both listed as "Not yet defined" | `docs/threat-model.md §T7` | T7 | Open | |
| H6 | OAuth2 token revocation procedure per provider not documented — no runbook for revoking a leaked Gmail or Outlook token | `docs/threat-model.md §T3` | T3 | Open | |
| H7 | rclone and rsnapshot Ofelia schedule intervals never defined — operations.md references them but gives no interval | `docs/operations.md §3` | — | Open | |
| H8 | B2/R2 bucket public-access block procedure not documented — required by T8 but no step-by-step in any doc | `docs/threat-model.md §T8` | T8 | Open | |
| H9 | CI `validate-configs` job does not lint Compose file for `0.0.0.0` port bindings on observability service ports — currently only checks YAML validity | `docs/cicd.md §validate-configs` | T17 | Open | |
| H10 | Alloy container Linux capabilities not specified — `cap_drop: ALL` should be set but is absent from docs | `docs/threat-model.md §T16` | T16 | Open | |
| H11 | Promtail reached end-of-life on March 2, 2026 — replaced with Grafana Alloy (official successor) across all docs | `docs/observability.md §Stack Decision` | T16 | Resolved | `docs/observability.md`, `docs/tech-stack.md`, `docs/threat-model.md` |

---

## Medium — Resolve Before the Relevant Implementation Task Begins

| ID | Item | Source | Status | Resolved in |
|---|---|---|---|---|
| M1 | Grafana dashboard layout not defined — deferred to implementation in both tech-stack.md and observability.md | `docs/observability.md §Open Decisions` | Open | |
| M2 | Prometheus retention period — "default 15 days is likely sufficient" but not confirmed; no config reference | `docs/observability.md §Open Decisions` | Open | |
| M3 | Loki retention period — similarly deferred; no config reference | `docs/observability.md §Open Decisions` | Open | |
| M4 | Container resource limits have no baseline values — `mem_limit` and `cpus` deferred to "implementation time based on observed usage" | `docs/tech-stack.md §Container resource limits` | Open | |
| M5 | Worker cycle interval (default 6h) not tied to a config variable — unclear how to override without code change | `docs/architecture.md §2` | Open | |
| M6 | mbsync schedule (15 min) mentioned in multiple docs but not tied to an Ofelia config variable | `docs/architecture.md §4` | Open | |
| M7 | Deploy SSH username never defined — `VPS_USER` secret is listed in cicd.md but the username itself is unspecified | `docs/cicd.md §GitHub Secrets` | Open | |
| M8 | Alertmanager "low-priority Slack channel or thread" for Info alerts — channel name or structure not defined | `docs/observability.md §Alert Routing` | Open | |
| M9 | Tang server uptime monitoring — how and where to alert on Tang becoming unreachable before a planned restart | `docs/threat-model.md §T15` | Open | |
| M10 | B2/R2 Object Lock (WORM): decision explicitly deferred in threat model | `docs/threat-model.md §T8` | Open | |
| M11 | Docker Content Trust / Cosign signature verification for custom images: decision deferred | `docs/threat-model.md §T5` | Open | |
| M12 | Third-party GitHub Actions audit process: "Not yet defined" | `docs/threat-model.md §T6` | Open | |
| M13 | Backup-server rsnapshot path permissions not defined — ciphertext directory and plaintext rsnapshot target should be restricted to a dedicated rsnapshot user/service account | `docs/threat-model.md §T9` | Open | |
| M14 | Backup-server host access policy not fully defined — T9 requires Tailscale-scoped administration, but no dedicated backup-server exposure/firewall baseline is documented | `docs/threat-model.md §T9` | Open | |

---

## Low — Ambiguous Specifications or Nice-to-Have Clarifications

| ID | Item | Source | Status | Resolved in |
|---|---|---|---|---|
| L1 | `AuthFailure` alert: exact log patterns/regex for mbsync auth errors not specified — too vague to implement | `docs/observability.md §Alert Mapping` | Open | |
| L2 | `BackupSyncStale` threshold (2h) assumes rclone runs more often than every 2h, but rclone schedule is undefined (M7 above creates circular dependency) | `docs/observability.md §Alert Mapping` | Open | |
| L3 | Pushgateway metric TTL not specified — by default Pushgateway never expires pushed metrics; a dead container's last-pushed values persist indefinitely | `docs/tech-stack.md §Pushgateway` | Open | |
| L4 | Deploy job "wait 30 seconds" health check delay is arbitrary — should be based on Docker health check probes, not a fixed sleep | `docs/cicd.md §deploy` | Open | |
| L5 | Rollback via manual `docker-compose.yml` edit on VPS is git-untracked — creates drift between repo and running config | `docs/cicd.md §Rollback` | Open | |
| L6 | Secret store mechanism ambiguous — architecture.md §5 describes both age-encrypted files and HashiCorp Vault as options without choosing one | `docs/architecture.md §5` | Open | |
| L7 | Pushgateway `--web.enable-admin-api=false`: "Not yet defined" in threat model | `docs/threat-model.md §T11` | Open | |
| L8 | Periodic OAuth2 token rotation cadence not specified — "re-authorize on a schedule" but no schedule given | `docs/threat-model.md §T3` | Open | |
| L9 | Grafana Tailscale SSO option mentioned but no decision made | `docs/observability.md §Access Controls` | Open | |

---

## Missing Procedures

Items that are referenced in docs as if they exist, or that are clearly needed but not yet written.

| ID | Item | Referenced in | Status | Resolved in |
|---|---|---|---|---|
| P1 | `python scripts/oauth2_setup.py` — referenced in operations.md as if it exists; not documented or implemented | `docs/operations.md §1` | Open | |
| P2 | Manifest DB rebuild maintenance script — mentioned in recovery.md Fallback as "implemented at implementation time" | `docs/recovery.md §Fallback` | Open | |
| P3 | mbsync UID cache corruption recovery — a common Maildir ingest failure mode with no documented resolution procedure | — | Open | |
| P4 | Recovery pre-condition check: explicit step to verify gocryptfs is mounted before `rclone copy` begins | `docs/recovery.md` (T18) | Resolved | `docs/recovery.md §Step 3` |
| P5 | Tailscale admin console monitoring for unexpected nodes — T7 flags this as a control but no procedure exists | `docs/threat-model.md §T7` | Open | |
