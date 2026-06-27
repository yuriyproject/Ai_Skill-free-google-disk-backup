# Example Backup Handoff

This is an example. It contains no real secrets.

## Project

- Project name: ExampleProject
- Server: `example-prod`
- Backup owner: project owner
- Backup system: Restic + rclone + Google Drive + systemd + Telegram

## What Is Backed Up

- PostgreSQL logical dump: `pg_dump -Fc`
- Production `.env`
- User uploads: `/opt/example/uploads`
- OCR images: `/opt/example/ocr`
- Deploy files:
  - `/opt/example/docker-compose.yml`
  - `/opt/example/Caddyfile`
- Source metadata:
  - Git commit
  - Git bundle
- `manifest.json`
- `checksums.sha256`

## What Is Excluded

- Docker images
- `node_modules`
- Python virtualenv
- Redis cache
- generated frontend build
- logs and temporary files

## Storage

- Restic repository: `rclone:gdrive:ExampleProject-Backups/restic`
- Google Drive folder: `ExampleProject-Backups`
- rclone remote name: `gdrive`
- OAuth scope: `https://www.googleapis.com/auth/drive.file`

## Secrets And Config Paths

Do not paste secret values into this document.

- Restic password file: `/etc/example-backup/restic-password`
- rclone config: `/etc/example-backup/rclone.conf`
- backup config: `/etc/example-backup/backup.env`
- Telegram backup bot token: stored in `/etc/example-backup/backup.env`

Critical rule:

```text
Save the Restic password in a password manager.
Without it, Google Drive backups cannot be decrypted.
```

## Schedule

- Backup: hourly
- Daily summary: daily
- Maintenance/prune/check: weekly
- Full repository check: monthly
- Restore test: monthly

Retention:

- hourly: 48 hours
- daily: 30 days
- weekly: 12 weeks
- monthly: 12 months

## systemd Units

- `example-backup.timer`
- `example-backup-summary.timer`
- `example-backup-maintenance.timer`
- `example-backup-full-check.timer`
- `example-backup-restore-test.timer`

Check timers:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
```

## First Verified Snapshot

- Snapshot ID: `abc123example`
- Timestamp: `2026-06-19T00:00:00Z`
- Backup duration: `42s`
- Telegram notification: received
- Repository status: ok

## Latest Restore Test

- Snapshot ID: `abc123example`
- Timestamp: `2026-06-19T00:15:00Z`
- Mode: isolated restore test
- Database restored: yes
- Checksums verified: yes
- Result: ok

## Restore Commands

List snapshots:

```bash
sudo /opt/example/ops/backup/restore.sh list
```

Stage latest snapshot:

```bash
sudo /opt/example/ops/backup/restore.sh stage latest
```

Test latest snapshot:

```bash
sudo /opt/example/ops/backup/restore.sh test latest
```

Stage a fixed snapshot:

```bash
sudo /opt/example/ops/backup/restore.sh stage abc123example
```

## Production Restore Warning

Do not restore to production until:

- owner explicitly approves;
- exact snapshot ID is selected;
- isolated restore-test passed;
- current production state is protected if possible;
- rollback plan exists.

Never print `.env`, Restic password, Telegram token, OAuth client secret, or
database dump contents into chat.
