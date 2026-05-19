# oi-tracker backups (skeleton — DISABLED per Q7)

> **Status:** **NOT ACTIVATED** in v1. Per `docs/00_DECISIONS_LOG.md Q7`,
> backups are future-work. This directory holds the reference
> implementation so activation = configuration, not invention.

> **Spec:** `docs/13_OPERATIONS.md §9` (RPO/RTO targets, pg_dump strategy,
> WAL-G optional path, restore drill procedure, activation criteria).

## What's here

```
deploy/backups/
├── pg-dump.sh                          # nightly dump wrapper (flock + verify)
├── restore-drill.sh                    # quarterly restore drill (auto-pick latest)
├── oi-tracker-backup.cron.disabled     # crontab entries (NOT in /etc/cron.d/)
└── README.md                           # this file
```

| File | Purpose | When to use |
|---|---|---|
| `pg-dump.sh` | One-shot nightly dump → `/var/backups/oi-tracker/oi_tracker-<TS>.dump`. Verifies dump readability via `pg_restore --list`. Optional rclone off-host copy when `OI_TRACKER_BACKUP_REMOTE` env is set. | Run from cron (`oi-tracker-backup.cron.disabled`) or manually for ad-hoc snapshot. |
| `restore-drill.sh` | Restores latest dump into throwaway `oi_tracker_restore_test` DB, verifies critical tables (`alert_rules`, `instruments`, `settings`), enforces RTO budget (default 2h per `13 §9.1`), drops test DB. | Quarterly, or after any backup-related change. |
| `oi-tracker-backup.cron.disabled` | crontab entries: 04:00 UTC nightly dump + 05:00 UTC retention purge + (commented-out) restore drill. **`.disabled` suffix marks it as not-activated**. | Activation = drop suffix and copy to `/etc/cron.d/oi-tracker-backup`. |

## Targets (per 13 §9.1)

| Parameter | Target |
|---|---|
| RPO | ≤ 24 hours (pg_dump nightly) — drops to ≤ 1h with WAL-G optional |
| RTO | ≤ 2 hours (full restore + verification) |
| Acceptable loss | up to 24 h of `oi_samples` (regenerable via fresh polling), alert history |
| Unacceptable loss | `alert_rules` (user customisations), `settings`, `instruments` blacklists |

## Activation procedure

> Run with `sudo` in a maintenance window. Pre-flight: confirm Q7 is no
> longer the active decision (i.e. there's a follow-up F-record reversing
> it, or owner explicitly approves activation).

### 1. Storage path

```bash
sudo install -d -o postgres -g postgres -m 0700 /var/backups/oi-tracker
sudo install -d -o postgres -g postgres -m 0700 /var/log/oi-tracker
```

### 2. Manual smoke

```bash
# Force one nightly dump now to confirm script works.
sudo -u postgres bash /var/www/oi-tracker/deploy/backups/pg-dump.sh
ls -lh /var/backups/oi-tracker/
# Verify dump is restorable (without actually restoring).
sudo -u postgres pg_restore --list /var/backups/oi-tracker/oi_tracker-*.dump | head -20
```

### 3. (Optional) Off-host upload

Off-host is **strongly recommended** — local backups die with the host disk.

```bash
# Install rclone, configure remote (one-time interactive):
sudo apt install -y rclone
sudo -u postgres rclone config        # add `b2-personal` or similar

# Persist remote URI for the cron wrapper:
sudo install -m 0644 -o root -g root /dev/null /etc/default/oi-tracker-backup
sudo bash -c 'cat >>/etc/default/oi-tracker-backup' <<'EOF'
OI_TRACKER_BACKUP_REMOTE=b2-personal:oi-tracker-backups
EOF
```

(Note: `pg-dump.sh` reads `OI_TRACKER_BACKUP_REMOTE` from env. Cron's
`EnvironmentFile=` equivalent is `cat /etc/default/oi-tracker-backup` —
prepend a `. /etc/default/oi-tracker-backup;` to the cron command line
or wrap `pg-dump.sh` in a small shim that sources it.)

### 4. Activate cron

```bash
sudo install -m 0644 -o root -g root \
  /var/www/oi-tracker/deploy/backups/oi-tracker-backup.cron.disabled \
  /etc/cron.d/oi-tracker-backup

sudo systemctl restart cron
# Verify cron picked it up:
sudo grep -i oi-tracker-backup /var/log/syslog | tail
```

### 5. Optional: enable WAL-G for RPO ≤ 1h (13 §9.3)

WAL-G install + `archive_mode = on` requires PG cluster restart →
**maintenance window with neighbour notification** (`detector`,
`grach-ege`). Out of scope for this skeleton.

### 6. Schedule first restore drill

```bash
# Manual — run within 7 days of activation to verify end-to-end.
sudo -u postgres bash /var/www/oi-tracker/deploy/backups/restore-drill.sh
```

If drill PASS — uncomment the quarterly drill cron line in
`oi-tracker-backup.cron.disabled` before installing it.

## Deactivation

```bash
sudo rm /etc/cron.d/oi-tracker-backup
sudo systemctl restart cron
# Optionally purge accumulated dumps:
# sudo rm -rf /var/backups/oi-tracker/oi_tracker-*.dump
```

The skeleton in this repo stays — it's a reference, not state.

## Why the `.disabled` suffix on the cron file?

If we shipped the cron file in `/etc/cron.d/` directly (e.g. via a
hypothetical install script), every fresh deploy would auto-activate
backups. Q7 says we don't activate. The `.disabled` suffix:

1. Makes it impossible to install via blind `cp deploy/* /etc/cron.d/` —
   the file simply isn't picked up by cron with that suffix.
2. Forces the operator to consciously rename / install in step 4 above.
3. Gets caught by the `M5.B PR-A verify` gate (`grep -c 'disabled'
   deploy/backups/*.cron* ≥ 1`).

## Cross-references

- Spec: `docs/13_OPERATIONS.md §9` — full plan including WAL-G and
  RPO/RTO targets.
- Decision: `docs/00_DECISIONS_LOG.md Q7` — current "not activated" status.
- Activation criteria: `docs/13_OPERATIONS.md §9.5`.
- M5 DoD reference: `docs/16_ROADMAP.md §M5 DoD`.
