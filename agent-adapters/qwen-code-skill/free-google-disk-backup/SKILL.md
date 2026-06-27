---
name: free-google-disk-backup
description: Design, implement, verify, and document encrypted no-paid-storage backups to the user's existing Google Drive with Restic + rclone. Use for server projects that need hybrid retention, Telegram backup alerts, Google Drive API/rclone setup, Stage Restore Server testing, restore-first workflows, and handoff documentation. Never print secrets.
---

# Free Google Disk Backup

Build a production backup setup where Restic encrypts data locally, rclone
stores the encrypted repository in the user's existing Google Drive, systemd
runs the jobs, and Telegram reports backup/restore status.

Use this skill when the user asks to set up or review backups for a server
project, especially AI-first/vibe-coded projects where agents may accidentally
damage production data.

For the full human runbook, read `references/human-runbook.md` when needed.

## Safety Rules

- Never paste secrets, `.env` contents, Restic passwords, OAuth secrets,
  `rclone.conf`, Telegram tokens, SSH keys, or database dumps into chat,
  GitHub, logs, or docs.
- Do not delete data, Docker volumes, Google Drive backups, databases, or old
  repositories unless the user explicitly approves a destructive action.
- Do not call backup complete until restore has been tested.
- Prefer a Stage Restore Server / restore-lab for off-production verification
  when the user is uncomfortable testing on production.
- Production restore must require explicit user approval, an exact snapshot ID,
  and a prior successful isolated restore-test.

## Target Architecture

Default stack:

- Restic for encrypted, deduplicated, compressed snapshots.
- rclone remote named `gdrive`.
- Google Drive OAuth scope: `https://www.googleapis.com/auth/drive.file`.
- Dedicated Google OAuth client, not rclone's shared public client.
- Restic repository: `rclone:gdrive:<ProjectName>-Backups/restic`.
- Backup scripts in `ops/backup/`.
- Root-only config in `/etc/<project>-backup/`.
- State in `/var/lib/<project>-backup/`.
- Restore staging in `/var/lib/<project>-restore/`.
- Telegram backup bot separate from the production app bot.

Recommended retention:

- hourly: 48 hours;
- daily: 30 days;
- weekly: 12 weeks;
- monthly: 12 months.

## Discovery

Before writing scripts, inspect:

1. database type and service/container name;
2. production `.env` or secrets files;
3. durable files: uploads, OCR/images, documents, media;
4. deploy files: `docker-compose.yml`, Caddy/nginx config;
5. source metadata: Git repo, branch, commit, bundle needs;
6. what is rebuildable and should not be backed up: Docker images, caches,
   dependencies, generated builds.

## Implementation Pattern

Create project-specific scripts:

- `ops/backup/lib/common.sh`: config, logging, locking, redaction, Telegram.
- `ops/backup/backup.sh`: dump DB, collect files, manifest, checksums, restic
  snapshot, notification.
- `ops/backup/restore.sh`: `list`, `stage`, `test`, guarded `production`.
- `ops/backup/maintenance.sh`: `restic forget --prune` and repository check.
- `ops/backup/daily-summary.sh`: latest status summary to Telegram.

Each backup should include:

- database logical dump;
- production `.env` or equivalent secrets file;
- durable user files;
- deploy files;
- Git bundle or exact source metadata;
- `manifest.json`;
- `checksums.sha256`.

## systemd Timers

Use timers instead of cron when possible:

- `<project>-backup.timer`: creates backup snapshots.
- `<project>-backup-summary.timer`: sends daily Telegram summary.
- `<project>-backup-maintenance.timer`: retention/prune/check.
- `<project>-backup-full-check.timer`: deeper repository check.
- `<project>-backup-restore-test.timer`: isolated restore-test.

Clarify timer purpose: backup frequency is not the same as maintenance or
restore-test frequency.

## Google Drive Setup

Guide the user through:

1. create/select Google Cloud project;
2. enable Google Drive API;
3. configure OAuth consent / Google Auth Platform;
4. add only `https://www.googleapis.com/auth/drive.file`;
5. create OAuth Client ID, type `Desktop app`;
6. configure rclone remote `gdrive`.

This should not use Google Cloud Storage, Compute Engine, Cloud SQL, BigQuery,
or any paid GCP service.

## Verification

Require evidence:

- `rclone lsd gdrive:` works;
- Restic repository initialized;
- manual backup succeeds;
- `restore.sh list` shows snapshots;
- `restore.sh stage latest` works;
- `restore.sh test latest` succeeds;
- state JSON records successful backup and restore-test;
- sensitive files are root-owned and mode `600`;
- config directories are mode `700`;
- timers are enabled and show sensible `NEXT` times;
- Telegram messages arrive.

## Stage Restore Server

For serious projects, prefer off-production restore proof:

1. use a separate VPS or persistent restore-lab;
2. install Restic, rclone, jq, git, Docker;
3. copy only required root-only backup config:
   - `restic-password`;
   - `rclone.conf`;
   - `backup.env`;
4. run `restic snapshots`;
5. restore latest snapshot into stage folder;
6. verify manifest/checksums;
7. restore database dump into an isolated container;
8. optionally boot the app with a stage-safe `.env`;
9. disable production Telegram bots, webhooks, payments, email/SMS, and external
   writes;
10. write restore-lab result to the handoff.

Never let a stage restore become a second production accidentally.

## Handoff

Leave a project-specific handoff document with:

- what is backed up and excluded;
- Restic repository path;
- retention schedule;
- systemd timer names;
- Telegram bot purpose;
- config/state paths;
- Restic password file path and warning to save it in a password manager;
- rclone config path;
- first verified snapshot ID;
- latest restore-test result;
- Stage Restore Server details, if configured;
- exact restore commands;
- warnings not to print secrets.
