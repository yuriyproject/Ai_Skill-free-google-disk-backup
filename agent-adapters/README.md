# Agent Adapters

This directory contains installable adapters for AI coding agents.

Use the adapter that matches your agent and do not mix formats:

```text
agent-adapters/
  codex-skill/
    free-google-disk-backup/        -> install into ~/.codex/skills/
  claude-code-skill/
    free-google-disk-backup/        -> install into ~/.claude/skills/
  qwen-code-skill/
    free-google-disk-backup/        -> install into ~/.qwen/skills/
  cursor-rules/
    .cursor/rules/free-google-disk-backup.mdc
```

The root README and `docs/` are for humans. These folders are for agent-side
installation.

`codex-skill`, `claude-code-skill`, and `qwen-code-skill` are real Skill-style
directories with `SKILL.md`.

`cursor-rules` is a Cursor Project Rule adapter. Cursor calls this feature
Rules, not Skills.

`research-needed.md` lists tools where no verified public adapter format is
included yet.

## Invocation

| Agent | Invoke |
| --- | --- |
| Codex | `Use $free-google-disk-backup.` |
| Claude Code | `/free-google-disk-backup` |
| Qwen Code | `Use free-google-disk-backup.` or browse `/skills` |
| Cursor | `Use the Free Google Disk Backup rule.` |

Do not put secrets inside skill folders.
