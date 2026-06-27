# Free Google Disk Backup: Claude Code Runbook

Use this runbook when applying `/free-google-disk-backup` in Claude Code.

## Recommended Prompt

```text
/free-google-disk-backup

Set up encrypted backups for this production project using Restic + rclone +
my existing Google Drive. Do not use paid Hetzner backup storage, Google Cloud
Storage, S3, or another paid backup SaaS. Do not delete anything without
explicit approval.

Use hybrid retention:
- hourly backups for 48 hours
- daily backups for 30 days
- weekly backups for 12 weeks
- monthly backups for 12 months

Back up the database, production .env/secrets, durable user files, deploy files,
and source metadata. Exclude caches, Docker images, dependencies, and generated
build artifacts unless they are business data.

Add Telegram notifications, daily summary, maintenance, restore-test, and a
Stage Restore Server plan. Leave a handoff document.
```

## What Claude Code Should Ask For

Only when needed:

- SSH access to the production server.
- GitHub access if scripts/docs should be committed.
- Google OAuth browser flow for rclone.
- Telegram BotFather token for a backup bot.
- Telegram chat ID.
- Optional Stage Restore Server access.

## Secret Handling

Never print or commit:

- Restic password;
- production `.env`;
- `rclone.conf`;
- OAuth client secret;
- Telegram bot token;
- SSH private key;
- database dump contents.

Use interactive prompts, root-only files, GitHub Secrets, or a password manager.

## Proof Of Work

Do not say the backup is working until:

- a manual backup succeeds;
- the latest snapshot is listed;
- an isolated restore-test succeeds;
- Telegram reports are verified;
- the Restic password location is documented;
- restore instructions are written.

For higher confidence, perform or document a Stage Restore Server drill.

