# Human Runbook For Qwen Code

Use this prompt in a Qwen Code session after installing the skill:

```text
Use free-google-disk-backup.

Set up encrypted regular backups for this production project to my existing
Google Drive using Restic + rclone. Do not use paid backup storage. Do not
delete anything without explicit approval. Do not print secrets.

I need hybrid retention:
- hourly backups for 48 hours
- daily backups for 30 days
- weekly backups for 12 weeks
- monthly backups for 12 months

Back up the database, production .env/secrets, durable user files, deploy files,
and source metadata. Add Telegram notifications, daily summary, maintenance,
restore-test, and a handoff document with restore instructions.
```

Before trusting the backup, ask Qwen Code to prove restore:

```text
Use free-google-disk-backup.

Verify the backup system without touching production data. Prefer Stage Restore
Server / restore-lab if available. Show which commands succeeded, where the
Restic password file lives, and how to restore a specific snapshot later.
```

Keep the Restic password in a password manager. The handoff should include only
the path to the password file, not the password value.
