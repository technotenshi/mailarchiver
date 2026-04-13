# mailarchiver — Operations Runbook

## Scope

This document covers routine day-to-day operations. For disaster recovery (full VPS loss), see `docs/recovery.md`.

---

## 1. Adding a New Email Account

### OAuth2 accounts (Gmail, Outlook)

1. Run the device authorization flow from a machine with a browser:
   - Gmail: `python scripts/oauth2_setup.py --provider gmail --account <name>`
   - Outlook: `python scripts/oauth2_setup.py --provider outlook --account <name>`
   - Follow the printed URL + device code prompt; the script writes the refresh token to stdout.

2. Create a Docker secret for the refresh token:
   ```
   echo '<refresh-token>' | docker secret create oauth2-<account> -
   ```

3. Add an `Account`, `Channel`, and `Group` block to the mbsync config for the new account. The account name used in the config becomes the Maildir namespace (`Maildir/<account>/`) and the `account` label on all metrics — keep it stable.

4. Add an Ofelia job entry for the new account's mbsync Channel.

5. Restart the affected containers:
   ```
   docker compose up -d --force-recreate mbsync worker
   ```

6. Verify the first ingest run succeeds: check Grafana for `IngestStale{account="<name>"}` to clear within the next scheduled run.

### App-password accounts (Yahoo, generic IMAP)

1. Generate an app-specific password in the provider's account settings.

2. Create a Docker secret:
   ```
   echo '<app-password>' | docker secret create apppassword-<account> -
   ```

3. Add mbsync config and Ofelia job as above.

4. Restart and verify as above.

---

## 2. Rotating OAuth2 Tokens or App Passwords

### 2a. Emergency token revocation (suspected compromise)

Use this procedure when a token or app password may have been leaked — for example, after a VPS breach, unexpected provider-side email access, or accidental credential logging. **Revoke on the provider side first**, then rotate the Docker secret using the steps below.

**Gmail**

1. Sign in to the account at `myaccount.google.com`.
2. Go to **Security → Third-party apps with account access**.
3. Find the mailarchiver / mbsync OAuth2 app entry and select **Remove Access**.
4. Confirm removal. The refresh token and all access tokens derived from it are immediately invalidated.

**Outlook / Microsoft (personal account)**

1. Sign in at `account.microsoft.com`.
2. Go to **Privacy → Apps and services that can access your data**.
3. Find the mailarchiver app and select **Remove these permissions**.

**Outlook / Microsoft (organizational / Azure AD account)**

1. An administrator must sign in to the Azure portal.
2. Navigate to **Azure Active Directory → Enterprise Applications**.
3. Find the mailarchiver app registration and select **Permissions → Revoke admin consent**, or disable the application.

**Yahoo (app-specific password)**

Yahoo uses app-specific passwords, not OAuth2 refresh tokens. Revocation is immediate once the app password is deleted:

1. Sign in to the Yahoo account.
2. Go to **Account Security → Generate and manage app passwords**.
3. Delete the mailarchiver entry.

**Verification**

After revoking on the provider side, confirm the old credential is dead before creating a new one:

```
docker compose exec mbsync mbsync --list <account>
```

The command should return an authentication error. If it still succeeds, the provider-side revocation has not yet propagated — wait a few minutes and retry.

**Next step:** proceed with the standard rotation steps below to issue a new credential and update the Docker secret.

---

### OAuth2 token refresh

1. Re-run the device authorization flow:
   ```
   python scripts/oauth2_setup.py --provider gmail --account <name>
   ```

2. Remove the old secret and create a new one:
   ```
   docker secret rm oauth2-<account>
   echo '<new-refresh-token>' | docker secret create oauth2-<account> -
   ```

3. Force-recreate the mbsync container to pick up the new secret:
   ```
   docker compose up -d --force-recreate mbsync
   ```

4. Verify the next scheduled mbsync run succeeds (check `IngestStale` alert clears in Grafana).

### App password rotation

Same as above: remove old secret, create new, force-recreate mbsync.

---

## 3. Manually Triggering Backup Verification

Run rclone sync + check against all three destinations outside the normal Ofelia schedule:

```
docker compose run --rm rclone
```

The rclone container runs sync then `rclone check`. On exit it pushes metrics to Pushgateway. The worker re-evaluates `backup_verifications` on its next cycle.

To check results immediately without waiting for the next worker cycle:

```
docker compose exec worker python -c "
import sqlite3, datetime
conn = sqlite3.connect('/manifest-db/manifest.db')
rows = conn.execute('''
  SELECT destination, status, confirmed_at
  FROM backup_verifications
  ORDER BY confirmed_at DESC LIMIT 10
''').fetchall()
for r in rows: print(r)
"
```

Or check the Grafana metric `mailarchiver_backup_last_verified_timestamp_seconds{destination}`.

### Local rsnapshot copy

rsnapshot runs on the primary VPS via host cron. It snapshots the gocryptfs ciphertext directories (`maildir-cipher` and `manifest-db-cipher`) directly — no separate mount or unlock step is required. The snapshot tree is always ciphertext; no plaintext is written to the snapshot directory. See `docs/architecture.md §Local rsnapshot backup` for the retention schedule and recovery procedure.

---

## 4. System Health Check

Run after a planned restart or as a routine check:

```
docker compose ps
```

All long-running containers should be in `running` state.

Check:
- Grafana: no active alerts on the main dashboard
- healthchecks.io: both heartbeats (worker + mbsync) green
- `mailarchiver_worker_halt` gauge = 0 (visible in Grafana or via `curl` to the worker metrics endpoint)
- No containers in `restarting` state (indicates a crash loop)

---

## 5. Planned VPS Restart

Before rebooting:

1. Verify the Tang server (secondary VPS) is reachable over Tailscale:
   ```
   tailscale ping <tang-vps-hostname>
   ```
   If Tang is unreachable, the gocryptfs mount will fail on reboot. Resolve Tang availability first.

2. Reboot the VPS. On restart:
   - `gocryptfs-mount.service` runs automatically (systemd, ordered `Before=docker.service`)
   - The systemd unit contacts Tang, derives the passphrase, and mounts `/maildir` and `/manifest-db`
   - Docker starts only after the unit exits successfully
   - `docker compose up` restores all containers

3. Verify with the health check above (§4).

**If Tang is unreachable after reboot:** the mount fails, Docker does not start. Use the manual fallback in §7 below.

---

## 6. Tang Key Rotation

Tang key rotation replaces the Clevis binding wrapper around the gocryptfs passphrase. The gocryptfs master key itself does not change — no re-encryption of data is needed.

The rotation procedure reads the passphrase from the **age-encrypted escrow file** (the same one used for the manual fallback in §7). The passphrase is never stored unencrypted on disk — only the age-encrypted copy exists outside of the Tang-derived in-memory derivation.

1. Decrypt the escrowed passphrase to use as input:
   ```
   age --decrypt -i <age-private-key-path> <escrowed-passphrase.age> > /tmp/gocryptfs-pass
   ```

2. Generate a new Tang binding using the passphrase:
   ```
   clevis encrypt tang '{"url": "http://<tang-vps-tailscale-ip>"}' \
     < /tmp/gocryptfs-pass \
     > /etc/mailarchiver/tang-binding-new.jwe
   shred /tmp/gocryptfs-pass
   ```

3. Test the new binding decrypts correctly:
   ```
   clevis decrypt < /etc/mailarchiver/tang-binding-new.jwe
   ```
   Confirm the output matches the escrowed passphrase.

4. Replace the active binding atomically:
   ```
   mv /etc/mailarchiver/tang-binding-new.jwe /etc/mailarchiver/tang-binding.jwe
   ```

5. On the Tang server: rotate the Tang signing key if desired (triggers a new ECDH keypair). After Tang key rotation, the old `tang-binding.jwe` is permanently unrecoverable from Tang — ensure the new binding is in place and tested first.

---

## 7. Manual gocryptfs Mount (Tang Fallback)

Use this procedure when Tang is unreachable and the VPS needs to start.

**Prerequisites:** Retrieve the gocryptfs passphrase from the escrowed age-encrypted file in the password manager.

1. Decrypt the escrowed passphrase:
   ```
   age --decrypt -i <age-private-key-path> <escrowed-passphrase.age>
   ```

2. Mount the Maildir volume:
   ```
   echo '<passphrase>' | gocryptfs -passfile /dev/stdin \
     /var/lib/mailarchiver/maildir-cipher \
     /var/lib/mailarchiver/maildir
   ```

3. Mount the manifest DB volume:
   ```
   echo '<passphrase>' | gocryptfs -passfile /dev/stdin \
     /var/lib/mailarchiver/manifest-db-cipher \
     /var/lib/mailarchiver/manifest-db
   ```

4. Start Docker Compose:
   ```
   docker compose up -d
   ```

5. Restore Tang connectivity as soon as possible. Once Tang is reachable again, verify that `gocryptfs-mount.service` succeeds on the next planned restart (i.e., normal automatic mounting resumes).

**Security note:** The passphrase is briefly present in the shell environment during step 2–3. Clear it from history after use (`history -c` or use a wrapper script that reads from a file and shreds it).
