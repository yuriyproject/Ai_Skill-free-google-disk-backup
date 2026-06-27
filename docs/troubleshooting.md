# Troubleshooting

This guide covers common problems when setting up Free Google Disk Backup.

## rclone OAuth Opens `127.0.0.1`

This is normal. rclone often starts a temporary local callback server during
OAuth.

If the browser cannot open it:

1. Use the browser on the same machine where rclone is running.
2. If rclone runs on a remote server, use rclone's manual/browser-assisted auth
   flow.
3. After successful auth, return to the terminal where `rclone config` is
   waiting.

Successful auth usually ends with:

```text
Success!
All done. Please go back to rclone.
```

## Google Says The User Is Not Eligible

If the OAuth app is in Testing mode, only test users can authorize it.

Check:

- the Gmail address is a real Google Account;
- the same account is used in the browser and in Google Drive;
- the account was added in Google Auth Platform / Audience / Test users;
- the correct Google Cloud project is selected.

If this is only for your own account, you can also publish the app to production
when using only approved/non-sensitive scopes, but do not add broader scopes
unless needed.

## `rclone lsd gdrive:` Fails

Check:

```bash
rclone config show gdrive
rclone lsd gdrive:
```

Do not paste the full config into chat. It may contain secrets.

Common causes:

- wrong remote name;
- OAuth token expired or revoked;
- Google Drive API not enabled;
- OAuth client deleted or changed;
- wrong Google account authorized.

## Restic Says Password Is Wrong

The Restic password is required to decrypt the repository. Google Drive cannot
help recover it.

Check:

```bash
sudo ls -l /etc/<project>-backup/restic-password
sudo wc -c /etc/<project>-backup/restic-password
```

Expected permissions:

```text
-rw------- 1 root root ... /etc/<project>-backup/restic-password
```

Do not rotate or overwrite this file unless you understand the Restic password
change process and have a verified restore path.

## Telegram Messages Do Not Arrive

Check:

- the backup bot token belongs to the backup bot, not the production bot;
- the target chat started the bot at least once;
- `TELEGRAM_BACKUP_CHAT_ID` is correct;
- outbound HTTPS works from the server;
- the script redacts token values in logs.

Safe test:

```bash
sudo /opt/<project>/ops/backup/daily-summary.sh
```

Do not print the token in chat or logs.

## Daily Summary Says `Maintenance: missing`

This is normal before the first maintenance timer has run.

Check timers:

```bash
sudo systemctl list-timers '*backup*' --all --no-pager
```

If the first weekly maintenance already passed, inspect:

```bash
sudo systemctl status <project>-backup-maintenance.timer --no-pager -l
sudo systemctl status <project>-backup-maintenance.service --no-pager -l
sudo journalctl -u <project>-backup-maintenance.service -n 100 --no-pager
```

## Restore Test Fails

Do not ignore this. Failed restore means the backup is not proven.

First checks:

1. Can Restic list snapshots?
2. Can Restic restore files into staging?
3. Does `checksums.sha256` pass?
4. Does the database dump exist?
5. Is the database restore command using the right engine/version?
6. Are stage credentials different from production credentials?

Try a fixed snapshot ID instead of `latest`:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
sudo /opt/<project>/ops/backup/restore.sh test <snapshot-id>
```

## Google Drive Quota Looks Strange

Restic deduplicates and compresses data, so repository growth may be much
smaller than raw project size.

If quota grows too fast:

- check that cache/build/dependency folders are excluded;
- check that OCR/media paths are intentional;
- verify retention and prune are running;
- inspect Restic snapshots and tags;
- confirm the backup is not nesting previous backup directories.

## Agent Wants To Print Secrets

Stop it. The correct behavior is to print paths and checksums, not secret
values.

Allowed:

```text
Restic password file: /etc/<project>-backup/restic-password
rclone config path: /etc/<project>-backup/rclone.conf
```

Not allowed:

```text
Restic password value
Telegram backup bot token value
OAuth client secret value
```
