# Cursor Rules Adapter

Cursor uses Project Rules, not Skills.

Install this adapter into a target project:

```text
agent-adapters/cursor-rules/.cursor/rules/free-google-disk-backup.mdc
  -> <project>/.cursor/rules/free-google-disk-backup.mdc
```

Then ask Cursor to use the Free Google Disk Backup rule while planning or
implementing backups.

Example:

```text
Use the Free Google Disk Backup rule.
```

Do not copy secrets into `.cursor/rules`.
