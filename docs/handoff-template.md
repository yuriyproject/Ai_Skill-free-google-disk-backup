# Handoff: Google Drive Backups For <ProjectName>

Last updated: YYYY-MM-DD

This document explains the production backup system for `<ProjectName>`.
Do not include secret values in this file.

## Current Status

- Server: `<ip-or-hostname>`
- Production path: `/opt/<project>`
- Backup tooling path: `/opt/<project>/ops/backup`
- Backup config directory: `/etc/<project>-backup`
- Backup state directory: `/var/lib/<project>-backup`
- Restore staging directory: `/var/lib/<project>-restore`
- Restic repository: `rclone:gdrive:<ProjectName>-Backups/restic`
- Telegram backup bot: `<bot-name>`
- Stage Restore Server: `<stage-ip-or-hostname>` or `not configured`

## What Is Backed Up

- Database dump: `<dump-type>`
- Production env/secrets file: `<path>`
- User uploads/media: `<paths>`
- Deploy files: `docker-compose.yml`, reverse proxy config
- Source metadata: Git bundle or commit metadata
- Manifest and checksums

## What Is Not Backed Up

- Redis/cache data
- Docker images
- Dependencies
- Build artifacts
- Temporary files

## Critical Secrets

Do not paste these values into chat, GitHub, logs, or docs.

- Restic password file:

```text
/etc/<project>-backup/restic-password
```

- rclone config:

```text
/etc/<project>-backup/rclone.conf
```

- backup config:

```text
/etc/<project>-backup/backup.env
```

Restic password must also be saved in the owner's password manager.
Without it, Google Drive backup files cannot be restored.

## Schedule

- Hourly backup: `<project>-backup.timer`
- Daily summary: `<project>-backup-summary.timer`
- Weekly maintenance: `<project>-backup-maintenance.timer`
- Monthly full check: `<project>-backup-full-check.timer`
- Monthly restore test: `<project>-backup-restore-test.timer`

Check timers:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
```

## Last Verified Results

- First verified backup snapshot: `<snapshot-id>`
- Last successful backup: `<timestamp>` / `<snapshot-id>`
- Last restore-test: `<timestamp>` / `<snapshot-id>`
- Last stage restore-lab check: `<timestamp>` / `<snapshot-id>` / `<level>` or `not configured`
- Last maintenance: `<timestamp>` or `not run yet`

## Normal Operations

List snapshots:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
```

Run isolated restore-test:

```bash
sudo /opt/<project>/ops/backup/restore.sh test latest
```

Check latest state:

```bash
sudo cat /var/lib/<project>-backup/state/latest-backup.json
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
```

## Production Restore

Production restore is intentionally guarded.

Rules:

- Run isolated restore-test first.
- Use an explicit snapshot ID.
- Get explicit owner approval.
- Do not print secrets.
- Create a fresh backup before changing production.

Example shape:

```bash
SNAPSHOT_ID="<snapshot-id>"

sudo /opt/<project>/ops/backup/restore.sh test "$SNAPSHOT_ID"

sudo RESTORE_CONFIRMATION=RESTORE_<PROJECT>_PRODUCTION \
  /opt/<project>/ops/backup/restore.sh production \
  --snapshot "$SNAPSHOT_ID" \
  --target /opt/<project>
```

## Stage Restore Server

Preferred restore proof happens outside production.

- Stage server: `<stage-ip-or-hostname>`
- Stage backup config: `/etc/<project>-backup`
- Stage restore root: `/var/lib/<project>-restore`
- Restore level used: `Level 1/2/3/4/5`
- App stack booted on stage: `yes/no`
- External integrations disabled: `Telegram/webhooks/payments/email/etc.`

Reference workflow:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
```

If app stack is booted on stage, use stage-safe env, not production `.env`.

## Notes For Future AI Agents

- Do not delete backups unless the owner explicitly asks.
- Do not run destructive Docker or database commands casually.
- Do not print `.env`, Restic password, rclone config, or tokens.
- Do not call backup complete until restore-test passes.
- If `Maintenance: missing` appears before the first weekly maintenance, it can be normal.
