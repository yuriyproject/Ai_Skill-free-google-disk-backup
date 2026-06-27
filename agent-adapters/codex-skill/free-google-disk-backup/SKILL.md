---
name: free-google-disk-backup
description: Design, implement, verify, and document a no-paid-storage backup system that stores encrypted project backups on the user's existing Google Drive using Restic + rclone. Use when the user wants regular hybrid-depth backups for a server project, easy restore, Telegram backup notifications, Google Drive API/rclone setup, Restic retention, systemd timers, or a reusable backup runbook without using paid Hetzner storage or paid Google Cloud services.
---

# Free Google Disk Backup

Use this skill to build a production backup setup where the user pays for no new
backup storage: Restic encrypts data locally, rclone uploads the encrypted
repository to the user's existing Google Drive, systemd schedules the jobs, and
Telegram reports success/failure.

For a human-facing checklist and runbook template, read
`references/human-runbook.md`.

## Non-Negotiables

- Never paste secrets, tokens, `.env` contents, Restic passwords, OAuth client
  secrets, rclone configs, or SSH keys into chat, logs, GitHub, or docs.
- Do not delete local data, Google Drive backups, Docker volumes, databases, or
  old repositories unless the user explicitly approves a destructive action.
- Do not assume the backup works because upload succeeded. Verify restore.
- Prefer `drive.file` OAuth scope for rclone so the app can access only files it
  creates/uses, not the whole Google Drive.
- Use a dedicated Google OAuth client for this backup, not rclone's shared client.
- Keep the Restic password outside the repo, ideally in both the server file and
  the user's password manager.

## Target Architecture

Default stack:

- `restic` for encrypted, deduplicated, compressed backups.
- `rclone` remote named `gdrive` pointing to Google Drive.
- Restic repo such as `rclone:gdrive:<ProjectName>-Backups/restic`.
- Server config in `/etc/<project>-backup/`, mode `700`.
- State in `/var/lib/<project>-backup/`.
- Restore staging in `/var/lib/<project>-restore/`.
- Project-local scripts under `ops/backup/`.
- systemd services/timers for backup, summary, maintenance, full check, and
  restore test.
- Separate Telegram bot for backup notifications.

Recommended hybrid retention:

- Hourly snapshots retained for 48 hours.
- Daily snapshots retained for 30 days.
- Weekly snapshots retained for 12 weeks.
- Monthly snapshots retained for 12 months.

## Discovery Workflow

Before designing scripts, inspect the project and server shape:

1. Identify the production data sources:
   - database type and container/service name;
   - `.env` or secret files;
   - user-uploaded files, OCR/images, documents, or media volumes;
   - source repository and deploy files (`docker-compose.yml`, reverse proxy).
2. Identify what not to back up:
   - rebuildable Docker images;
   - dependency folders;
   - Redis/cache data unless it contains durable business data;
   - generated frontend builds.
3. Determine safe restore targets:
   - isolated test restore;
   - separate Stage Restore Server / restore-lab for off-production testing;
   - staging extraction;
   - production restore only with explicit confirmation.
4. Confirm that the user wants Google Drive as the storage target and accepts
   rclone OAuth.

## Implementation Pattern

Create a project-specific `ops/backup/` folder with these scripts:

- `lib/common.sh`: config loading, logging, locking, secret redaction, Telegram.
- `backup.sh`: create logical dump/files bundle, manifest, checksums, restic
  backup, Telegram result.
- `restore.sh`: `list`, `stage`, `test`, and guarded `production` modes.
- `maintenance.sh`: `restic forget --prune` with hybrid retention and a check.
- `daily-summary.sh`: Telegram daily summary from state files and repository
  health.

Prefer logical database backups:

- PostgreSQL: `pg_dump -Fc`.
- MySQL/MariaDB: `mysqldump` or `mariadb-dump` with consistent options.
- SQLite: copy via `.backup` or application-safe snapshot.

Each backup should include:

- database dump;
- production `.env` or equivalent secrets file;
- durable uploaded/user files;
- deploy files (`docker-compose.yml`, reverse proxy config);
- source Git bundle or exact Git revision metadata;
- `manifest.json`;
- `checksums.sha256`.

## systemd Schedule

Use timers instead of cron when possible:

- `project-backup.timer`: hourly backup.
- `project-backup-summary.timer`: daily Telegram summary.
- `project-backup-maintenance.timer`: weekly retention/prune/check.
- `project-backup-full-check.timer`: monthly deeper Restic check.
- `project-backup-restore-test.timer`: monthly isolated restore test.

When interpreting daily summaries, `Maintenance: missing` is expected until the
first weekly maintenance run creates its state file.

## Google Setup

Guide the user through:

1. Create or choose a Google Cloud project.
2. Enable Google Drive API.
3. Configure OAuth consent.
4. Add only the Google Drive API scope:
   `https://www.googleapis.com/auth/drive.file`.
5. Create an OAuth Client ID of type `Desktop app`.
6. Configure rclone on the server or with browser-assisted auth.

Clarify that this uses the user's existing Google Drive storage. It should not
use Google Cloud Storage, Compute Engine, Cloud SQL, or other paid GCP products.
Suggest a low billing budget alert for peace of mind, but do not require paid
services.

## Verification Checklist

Do not call the setup complete until all relevant checks pass:

- Manual backup succeeds.
- Latest Restic snapshot is listed.
- Daily summary can report latest backup and repository status.
- Isolated restore test succeeds.
- For higher-safety projects, a Stage Restore Server or temporary test VPS can
  restore the snapshot away from production.
- Restored database dump can be inspected.
- Telegram success/failure notification works.
- Sensitive files are owned by root and mode `600`; config directory mode `700`.
- systemd timers are enabled and have sensible `NEXT` times.
- Handoff/runbook documents where the Restic password and rclone config live.

## Restore Safety

Default restore flow:

1. `restore.sh list`
2. `restore.sh stage latest`
3. verify `checksums.sha256`
4. run isolated `restore.sh test <snapshot>`
5. only then consider production restore

Production restore must require an explicit confirmation environment variable or
token and an explicit snapshot ID. Do not allow casual `latest` production
restore.

For projects where testing on production is uncomfortable, prefer a separate
Stage Restore Server: install Restic/rclone/Docker there, copy only the required
root-only backup config, restore latest snapshot, restore the database into an
isolated container, and optionally boot the app with a stage-safe `.env` that
disables production Telegram bots, webhooks, payments, and external writes.

## Documentation To Leave Behind

Create or update a project handoff document with:

- backup architecture;
- exact repository path;
- schedule and retention;
- secret file locations;
- restoration procedure;
- what was verified;
- first known snapshot ID;
- Telegram bot purpose;
- warning not to lose the Restic password.

Keep docs free of secret values.
