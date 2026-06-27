# Research Needed

These tools are intentionally not shipped with installable adapters yet because
this repository should not invent formats that are not verified.

| Tool | Current status | What to do for now |
| --- | --- | --- |
| Grok Build | No verified public Skill/Rules-style adapter format was found. Some users report Claude Code skill compatibility, but this repository does not treat that as official until documented. | If your Grok Build terminal reads Claude Code skills, install/use `agent-adapters/claude-code-skill/free-google-disk-backup`. Otherwise use the root README as a prompt/runbook until official docs describe an installable agent instruction format. |
| Google Antigravity | The official public site exists, but no verified Skill/Rules-style adapter format was found. | Use the root README manually, or add an adapter later when official docs describe the format. |
| Z.ai / GLM | Z.ai provides models and coding capabilities, but no verified installable Skill/Rules adapter format was found. | Use the root README manually, or add an adapter later when official docs describe the format. |

If official documentation appears later, add a new adapter folder with a clear
name, for example `grok-build-rules` or `antigravity-rules`, and include a link
to the source documentation in that adapter README.
