# Restore Drill

Run a restore drill regularly. Backup upload is not enough.

Recommended cadence:

- monthly for normal projects;
- before risky migrations or large AI-agent refactors;
- after changing backup scripts, retention, rclone config, or database layout.

## Preferred Drill: Stage Restore Server

Use a separate VPS or restore-lab whenever possible.

High-level flow:

1. Prepare a clean stage server.
2. Install required tools: Restic, rclone, jq, git, Docker.
3. Copy only required backup config:
   - `/etc/<project>-backup/restic-password`;
   - `rclone.conf`;
   - `/etc/<project>-backup/backup.env`.
4. Verify repository access.
5. Restore the selected snapshot into a stage folder.
6. Verify `manifest.json` and `checksums.sha256`.
7. Restore the database dump into an isolated database container.
8. Optionally boot the app with a stage-safe `.env`.
9. Disable production webhooks, bots, payments, email/SMS, and external writes.
10. Record the result in the project handoff.

## Commands

List snapshots:

```bash
sudo /opt/<project>/ops/backup/restore.sh list
```

Stage latest snapshot:

```bash
sudo /opt/<project>/ops/backup/restore.sh stage latest
```

Run isolated restore test:

```bash
sudo /opt/<project>/ops/backup/restore.sh test latest
```

For serious restores, prefer a fixed snapshot ID:

```bash
sudo /opt/<project>/ops/backup/restore.sh stage <snapshot-id>
sudo /opt/<project>/ops/backup/restore.sh test <snapshot-id>
```

## Expected Evidence

Record:

- snapshot ID;
- restore drill timestamp;
- restore duration;
- database restore result;
- key tables/files inspected;
- location of staged restore;
- whether checksums passed;
- whether the app was booted in stage mode;
- who reviewed the result.

## Failure Handling

If restore fails:

1. Do not delete the broken repository.
2. Save logs.
3. Check Restic password path.
4. Check rclone config and Google OAuth status.
5. Try an older snapshot.
6. Run `restic check` or the project maintenance script.
7. Fix the root cause before trusting future backups.

## Production Restore Rule

Production restore requires:

- explicit owner approval;
- fixed snapshot ID;
- successful isolated restore-test;
- backup of current production state if possible;
- clear rollback plan;
- confirmation token if the script supports it.

Do not production-restore `latest` casually.
