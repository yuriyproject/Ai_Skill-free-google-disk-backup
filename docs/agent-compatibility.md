# Agent Compatibility

This project ships adapters only for formats that are known or intentionally
scoped.

| Code system | Support | Adapter | Notes |
| --- | --- | --- | --- |
| Codex | Supported | `agent-adapters/codex-skill/free-google-disk-backup` | Native Skill-style folder with `SKILL.md` and `agents/openai.yaml`. |
| Claude Code | Supported | `agent-adapters/claude-code-skill/free-google-disk-backup` | Claude Code skill folder with `SKILL.md`. |
| Qwen Code | Supported | `agent-adapters/qwen-code-skill/free-google-disk-backup` | Qwen Code Agent Skill folder with `SKILL.md`. |
| Cursor | Supported as Rules | `agent-adapters/cursor-rules/.cursor/rules/free-google-disk-backup.mdc` | Cursor uses Project Rules, not Skills. |
| Grok Build | Conditional | Use Claude Code adapter only if Grok Build loads Claude Code skills | No separate adapter until official docs define a different format. |
| Google Antigravity | Research needed | None | Use README manually until official installable instruction format is confirmed. |
| Z.ai / GLM | Research needed | None | Use README manually until official installable instruction format is confirmed. |

## Invocation

| Code system | Invocation |
| --- | --- |
| Codex | `Use $free-google-disk-backup.` |
| Claude Code | `/free-google-disk-backup` |
| Qwen Code | `Use free-google-disk-backup.` or `/skills` |
| Cursor | `Use the Free Google Disk Backup rule.` |
| Grok Build | `/free-google-disk-backup` only when Claude Code skill compatibility is enabled |

## Rule For New Adapters

Do not add an adapter just because a model exists.

Add a new adapter only when at least one is true:

- official documentation describes the installable instruction format;
- the tool explicitly supports another existing format, such as Claude Code
  skills;
- the adapter is clearly labelled as experimental and does not pretend to be
  official.

If the format is not verified, document it in
`agent-adapters/research-needed.md`.
