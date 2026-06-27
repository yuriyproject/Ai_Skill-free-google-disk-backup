# Publishing Checklist

Use this before publishing the repository publicly.

## Secret Safety

- [ ] No real `.env` files.
- [ ] No Restic passwords.
- [ ] No Telegram bot tokens.
- [ ] No OAuth client secrets.
- [ ] No `rclone.conf` with tokens.
- [ ] No SSH private keys.
- [ ] No production hostnames or IPs unless intentionally public.
- [ ] Example files use placeholders only.

Suggested scan. Keep exact secret signatures in your local shell history,
pre-commit hook, or security tool config rather than committing noisy patterns
to this repository:

```bash
grep -RniE "<your-secret-patterns>" .
```

## Documentation

- [ ] README explains the goal and quick start.
- [ ] Compatibility Matrix is current.
- [ ] Restore-first rule is visible.
- [ ] Restic password warning is visible.
- [ ] Google Drive API setup is clear.
- [ ] Stage Restore Server option is documented.
- [ ] Troubleshooting guide exists.
- [ ] Handoff template and example are present.

## Agent Adapters

- [ ] Codex skill has `SKILL.md`.
- [ ] Claude Code skill has `SKILL.md`.
- [ ] Qwen Code skill has `SKILL.md`.
- [ ] Cursor rule uses `.cursor/rules/*.mdc`.
- [ ] Unsupported tools are listed in `agent-adapters/research-needed.md`.
- [ ] No adapter claims support without a verified format.

## Examples

- [ ] systemd examples contain placeholders, not real project names.
- [ ] prompt examples do not include secrets.
- [ ] handoff example contains fake values only.
- [ ] config examples use placeholders only.

## Final Review

- [ ] License is present.
- [ ] SECURITY.md explains how to report issues.
- [ ] CONTRIBUTING.md explains secret hygiene.
- [ ] CHANGELOG.md has initial entry.
- [ ] ROADMAP.md describes future scope without overpromising.
- [ ] GitHub issue and PR templates are present.

After publishing, create the first release tag only after reading the repository
as if you were a new user.
