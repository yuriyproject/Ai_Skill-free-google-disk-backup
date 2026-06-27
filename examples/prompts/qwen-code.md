# Qwen Code Prompt

Use this after installing:

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

Do not call the setup complete until restore has been tested.
```
