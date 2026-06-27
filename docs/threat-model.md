# Threat Model

Free Google Disk Backup is designed as a safety layer for AI-first development
and small production projects.

## Assets

- Production database.
- Production `.env` and secrets.
- User-uploaded files.
- OCR/images/media/documents.
- Deploy configuration.
- Source metadata needed for restore.
- Restic password.
- rclone config and Google OAuth refresh token.

## Main risks

### AI-agent mistakes

An agent can accidentally:

- delete files;
- run the wrong deploy command;
- break migrations;
- overwrite config;
- remove production data;
- make a bad change that is noticed weeks later.

Mitigation:

- encrypted off-server backups;
- hybrid retention;
- restore-test;
- Stage Restore Server / restore-lab for off-production verification;
- handoff document.

### Server loss

The VPS can be lost or corrupted.

Mitigation:

- Google Drive repository outside the server;
- restore instructions for a new server;
- source metadata and deploy files in backup.

### Secret leakage

Backup systems concentrate sensitive data.

Mitigation:

- root-only config directory;
- file mode `600` for secrets;
- no secrets in Git;
- no secrets in AI chat;
- dedicated Telegram backup bot;
- dedicated Google OAuth client.

### False sense of safety

An upload-only backup can be unusable.

Mitigation:

- isolated restore-test;
- checksums;
- manifest;
- periodic maintenance and full checks.

## Out of scope

This project does not fully protect against:

- compromised Google account;
- lost Restic password;
- compromised server before backup is created;
- malicious root user;
- legal/compliance archiving requirements;
- zero-downtime enterprise disaster recovery.

## Core security rule

Backup is only useful when it is:

- encrypted;
- off-server;
- regularly created;
- retained with enough history;
- tested through restore;
- preferably tested away from production for serious projects;
- documented well enough for another person or agent to recover it.
