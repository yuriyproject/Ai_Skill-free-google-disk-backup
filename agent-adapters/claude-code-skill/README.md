# Claude Code Skill

This folder contains the Claude Code-compatible skill adapter.

Install this folder:

```text
agent-adapters/claude-code-skill/free-google-disk-backup
```

to:

```text
~/.claude/skills/free-google-disk-backup
```

Then invoke it in Claude Code:

```text
/free-google-disk-backup
```

If another terminal agent explicitly supports Claude Code skills, use this same
folder and invocation unless that agent documents a different command.

Prompt for a new Claude Code chat:

```text
Install the Claude Code skill from this repository:
<GITHUB_REPO_URL>

Use agent-adapters/claude-code-skill/free-google-disk-backup and copy it to
~/.claude/skills/free-google-disk-backup. Do not copy secrets.
```

Project-local install:

```text
mkdir -p .claude/skills
cp -r /path/to/free-google-disk-backup/agent-adapters/claude-code-skill/free-google-disk-backup .claude/skills/
```

Review all project skills before trusting a repository.
