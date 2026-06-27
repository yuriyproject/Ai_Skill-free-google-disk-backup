# Free Google Disk Backup: Human Runbook

This runbook explains how to ask an AI agent to build the same style of backup
system for another project.

## What This Backup Model Does

The system creates encrypted backups on the production server and stores them in
the user's existing Google Drive. The core tools are:

- Restic: encrypts, deduplicates, compresses, lists, verifies, and restores.
- rclone: connects Restic to Google Drive.
- systemd timers: run backups and checks automatically.
- Telegram bot: sends success, failure, and daily summary messages.

The backup is free in the practical sense: it uses existing Google Drive space
and does not require Hetzner backup storage, Google Cloud Storage, S3, or paid
backup SaaS.

## Recommended Prompt For A New AI Chat

Use this when starting a new project backup session:

```text
Use $free-google-disk-backup.

I want a free Google Drive backup system for this production project.
Use Restic + rclone + my existing Google Drive. Do not use paid Hetzner backup
storage or paid Google Cloud storage. Do not delete anything.

I want hybrid retention:
- hourly backups for 48 hours
- daily backups for 30 days
- weekly backups for 12 weeks
- monthly backups for 12 months

Back up the database, production .env/secrets file, durable user-uploaded files,
deploy files, and enough source metadata to restore the project. Exclude caches,
Docker images, dependencies, and rebuildable artifacts unless they are truly
business data.

Add Telegram notifications and daily summaries. Make restore easy: list, stage,
test restore, and guarded production restore. Leave a handoff document that
explains how to recover the system and especially where the Restic password is.
```

## Accesses The Agent May Need

Provide only when the agent actually needs them:

- SSH access to the production server.
- GitHub access to the repository if scripts/docs should be committed.
- Google account access through an OAuth browser flow for rclone.
- Telegram BotFather token for a dedicated backup bot.
- Telegram chat ID for notifications.

Do not paste secrets directly into chat. Prefer interactive terminal prompts,
password manager sharing, environment files on the server, or GitHub Secrets
when appropriate.

## Google Setup Checklist

1. Create or select a Google Cloud project.
2. Enable Google Drive API.
3. Configure OAuth consent screen.
4. Add the scope:
   `https://www.googleapis.com/auth/drive.file`
5. Create OAuth Client ID:
   - Application type: Desktop app
   - Name: project backup or similar
6. Use rclone to authenticate Google Drive.

This setup should not use:

- Google Cloud Storage buckets
- Compute Engine
- Cloud SQL
- BigQuery
- paid backup products

For peace of mind, set a very small Google Cloud billing budget alert such as
`$1`. Budget alerts warn you; they do not stop services automatically.

## Telegram Setup Checklist

1. Create a dedicated bot in BotFather, for example `ProjectBackupBot`.
2. Start a chat with the bot.
3. Get the chat ID safely using `getUpdates` or another trusted method.
4. Store bot token and chat ID in the backup config file on the server.
5. Test one success message and one failure path if possible.

Use a separate backup bot instead of the production application bot. Backup
alerts are operational messages and should not be mixed with customer traffic.

## What Should Be Backed Up

Always consider:

- database logical dump;
- production `.env` or equivalent secrets file;
- uploaded files and durable media;
- OCR/images/documents that cannot be regenerated;
- `docker-compose.yml`;
- reverse proxy config such as `Caddyfile` or nginx config;
- Git bundle or exact Git commit/repository metadata;
- manifest and checksums.

Usually exclude:

- Redis cache and queue state;
- Docker images;
- dependency folders such as `node_modules` or virtualenvs;
- build artifacts that can be regenerated;
- logs unless there is a compliance reason.

## Expected Server Layout

Use project-specific names, but this shape is recommended:

```text
/opt/<project>/ops/backup/          backup scripts in the repository
/etc/<project>-backup/              root-owned config and secrets
/var/lib/<project>-backup/          state, staging, cache
/var/lib/<project>-restore/         staged restores
```

Sensitive files should be:

```text
owner: root
mode: 600
```

Sensitive config directories should be:

```text
owner: root
mode: 700
```

The Restic password file is critical. Without it, Google Drive backup files are
encrypted and cannot be restored.

## systemd Timers To Expect

Names vary by project, but expect:

```text
<project>-backup.timer              hourly backup
<project>-backup-summary.timer      daily Telegram summary
<project>-backup-maintenance.timer  weekly retention/prune/check
<project>-backup-full-check.timer   monthly deeper repository check
<project>-backup-restore-test.timer monthly isolated restore test
```

Useful read-only checks:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
sudo systemctl status <project>-backup.service --no-pager -l
sudo journalctl -u <project>-backup.service -n 100 --no-pager
```

## How To Interpret Telegram Summary

Example:

```text
Latest: <snapshot-id> (OK)
Timestamp: 2026-06-18T08:03:06Z
Duration: 41s
Repository: ok
Maintenance: missing
Restore test: ok
Google Drive used/free: 5063760/5496985770348
```

Meaning:

- `Latest (OK)`: newest backup succeeded.
- `Repository: ok`: Restic can read the Google Drive repository.
- `Maintenance: missing`: no weekly maintenance state file exists yet. This is
  normal before the first scheduled weekly maintenance run.
- `Restore test: ok`: an isolated restore test succeeded.
- `Google Drive used/free`: used and free bytes on the Drive account.

If `Maintenance: missing` remains after the first weekly maintenance date, check
the maintenance timer and journal logs.

## Proof The Setup Works

Ask the agent to prove these before calling the work complete:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test latest
sudo cat /var/lib/<project>-backup/state/latest-backup.json
sudo cat /var/lib/<project>-backup/state/latest-restore-test.json
```

The exact paths may differ. The agent should adapt them to the project.

## Restore Workflow

Safe default:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh stage latest
sudo /opt/<project>/ops/backup/restore.sh test <snapshot-id>
```

Production restore must be explicit and guarded. The agent should not run a
production restore without clear approval, an exact snapshot ID, and a prior
successful isolated restore test.

## Handoff Document Requirements

The agent should leave a project-specific handoff document with:

- backup architecture;
- repository path in Google Drive;
- retention schedule;
- systemd timer names;
- Telegram bot purpose;
- config/state paths;
- first verified backup snapshot ID;
- latest restore-test result;
- exact restore commands;
- warning not to lose the Restic password;
- reminder that no secret values should be committed or pasted into chat.

Good document names:

```text
HandOffBackupGdrive.md
BackupRunbook.md
docs/backup-google-drive.md
```

