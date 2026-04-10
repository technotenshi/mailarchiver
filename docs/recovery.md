# mailarchiver — Recovery Runbook

## Scope

This document covers full VPS recovery — the scenario where the primary server is destroyed or unrecoverable and must be rebuilt from scratch. Partial failure scenarios (backup destination unreachable, disk pressure, worker halted) are handled by the operational alerts in `docs/observability.md`.

---

## Before You Need This

Three prerequisites must be in place before a disaster occurs. If any are missing, full recovery may be impossible.

1. **rclone crypt passphrase is escrowed** — stored in a location independent of the VPS (personal password manager, printed copy in a secure location, or a second encrypted secret store). Without it, the B2 and R2 backups cannot be decrypted.

2. **The gocryptfs escrow mechanism is recoverable** — the `age` recipient used to encrypt the fallback passphrase, and access to the password-manager entry where that age-encrypted blob is stored, must both be recoverable. During a full VPS rebuild, the recovery procedure generates a new gocryptfs passphrase and immediately re-escrows it in the same age-encrypted format before any restore begins.

3. **Docker Compose config and secrets are version-controlled or backed up** — the Compose file, `gocryptfs-mount.service`, Ofelia config, Prometheus/Alertmanager config, and all Docker secrets must be recoverable independently of the VPS disk. Secrets (OAuth2 tokens, app passwords, webhook URLs) should be documented in a password manager; the configs may be in a private git repository.

---

## Recovery Procedure

### Step 1 — Retrieve credentials from escrow

Before provisioning anything:
- Retrieve the rclone crypt passphrase from its escrowed location
- Retrieve the `age` recipient or public key used to encrypt the gocryptfs fallback passphrase
- Retrieve all Docker secrets (OAuth2 tokens, app passwords, SMTP credentials, webhook URLs) from the password manager
- Confirm you have the Compose and config files (from git or backup), including `gocryptfs-mount.service`

### Step 2 — Provision a new VPS

- Same OS (Ubuntu), same architecture as the original
- Install Docker Engine and Docker Compose v2
- Install Tailscale and authenticate to the same tailnet (this restores connectivity to the local rsnapshot server and the Tang VPS)
- Install `gocryptfs`, `clevis-tang`, and `age`
- Verify Tang server (secondary VPS) is reachable over Tailscale before proceeding — all gocryptfs mounts depend on it
- If Tang is unreachable, stop here. Do not run `rclone copy` until Tang connectivity is restored and the host-level gocryptfs mounts are working.

### Step 3 — Recreate encrypted volumes, start the host mounts, and verify targets

Rebuild the encrypted-at-rest layer on the new VPS before restoring any data. Use the standard host paths from `docs/architecture.md §7` so the recovered host matches the steady-state deployment:

```
mkdir -p /etc/mailarchiver \
  /var/lib/mailarchiver/maildir-cipher \
  /var/lib/mailarchiver/manifest-db-cipher \
  /var/lib/mailarchiver/maildir \
  /var/lib/mailarchiver/manifest-db

umask 077
openssl rand -base64 32 > /tmp/gocryptfs-pass
age --encrypt -r <age-recipient> -o /tmp/gocryptfs-pass.age /tmp/gocryptfs-pass
```

Before continuing, store `/tmp/gocryptfs-pass.age` in the password manager or equivalent off-host escrow location. The rebuilt VPS uses this new passphrase; the previous host's fallback record no longer matches the recovered system.

Initialize both ciphertext directories with the new passphrase:

```
gocryptfs -init -passfile /tmp/gocryptfs-pass \
  /var/lib/mailarchiver/maildir-cipher
gocryptfs -init -passfile /tmp/gocryptfs-pass \
  /var/lib/mailarchiver/manifest-db-cipher

clevis encrypt tang '{"url":"http://<tang-vps-tailscale-ip>:7500"}' \
  < /tmp/gocryptfs-pass \
  > /etc/mailarchiver/tang-binding.jwe
```

Install the recovered `gocryptfs-mount.service` at `/etc/systemd/system/gocryptfs-mount.service`, make sure it references the newly generated `/etc/mailarchiver/tang-binding.jwe`, then start it before any restore:

```
systemctl daemon-reload
systemctl enable --now gocryptfs-mount.service
```

Verify both plaintext targets are backed by the matching `*-cipher` mount before restoring data:

```
findmnt -no SOURCE,TARGET,FSTYPE -T /var/lib/mailarchiver/maildir
findmnt -no SOURCE,TARGET,FSTYPE -T /var/lib/mailarchiver/manifest-db
```

Each command must show the corresponding `*-cipher` directory as `SOURCE`, the plaintext path as `TARGET`, and `fuse.gocryptfs` as `FSTYPE`. If either check fails, stop and fix the mount service before running `rclone copy`.

Once the mounts are verified and the encrypted escrow file is stored off-host, remove the temporary plaintext passphrase material:

```
shred -u /tmp/gocryptfs-pass
rm -f /tmp/gocryptfs-pass.age
```

### Step 4 — Restore Maildir from offsite backup

Choose B2 or R2 as the restore source (prefer whichever had the most recent confirmed sync, visible in Grafana — or use either if Grafana is unavailable).

```
# Configure rclone with the crypt remote pointing to B2 or R2.
# Restore only into the verified gocryptfs mount target:
rclone copy b2-crypt:maildir /var/lib/mailarchiver/maildir/
```

This is the longest step. Duration depends on Maildir size and network speed.

### Step 5 — Restore manifest DB

The `manifest-db` volume was synced alongside `maildir`. Restore it from the same backup source into the verified gocryptfs mount:

```
rclone copy b2-crypt:manifest-db /var/lib/mailarchiver/manifest-db/
```

If the manifest DB backup is absent or corrupt, skip to the fallback procedure below.

### Step 6 — Restore remaining configuration and secrets

- Place Docker Compose file and all remaining service configs in their expected paths
- Load Docker secrets from the password manager
- Verify rclone.conf contains the correct crypt remote configurations for B2 and R2
- Do not overwrite the newly generated `/etc/mailarchiver/tang-binding.jwe` with a stale copy from backup or version control

### Step 7 — Start all containers

```
docker compose up -d
```

Verify:
- Dovecot responds to IMAP connections from Tailscale peers
- Worker starts and transitions out of HALT state
- Grafana dashboards show expected state
- External heartbeat resumes (healthchecks.io shows green)

### Step 8 — Re-run backup verifications

After restore, all `backup_verifications` rows for the new VPS's rclone will be stale. Trigger a manual rclone sync + check run to repopulate confirmation timestamps before the deletion worker resumes normal operation:

```
docker compose run --rm rclone
```

---

## Fallback: Manifest DB Lost

If the manifest DB cannot be restored, the deletion worker cannot run until the manifest is rebuilt. The Maildir data is intact; only the tracking state is lost.

Rebuild procedure (implemented as a maintenance script at implementation time):

1. Scan the restored Maildir; register all messages into a fresh `messages` and `provider_copies` table
2. Set all `provider_copies.deletion_status` to `ELIGIBLE` with `ingested_at` set conservatively (e.g. current time minus 90 days, to start the retention clock fresh)
3. Run `rclone check` against all three destinations and populate `backup_verifications`
4. Resume normal worker operation

**Note:** After a manifest rebuild, the deletion worker will not delete any provider copies for at least 90 days (the retention window restarts). This is the safe and correct behavior.

---

## RPO / RTO Summary

| Metric | Value | Notes |
|---|---|---|
| **RPO** (data at risk during outage) | Near-zero | Email remains on provider until 90-day deletion eligibility; provider copies are the safety net during a VPS outage |
| **RTO** (time to full operation) | 1–4 hours | Dominated by rclone decrypt + restore speed from B2/R2; small Maildir restores in ~30 min, large ones may take hours |
| **Maximum safe outage with no data loss** | 90 days | Beyond this, some emails might be deleted from providers before being archived; external heartbeat must alert long before this threshold |
