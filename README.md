# Skills

[Claude Code](https://claude.com/claude-code) skills I use for my own homelab, written to work on anyone's setup.

| Skill | What it does |
|---|---|
| [`immich/upgrade`](immich/upgrade/SKILL.md) | Upgrades a self-hosted Immich (Docker Compose): reads every release in between for breaking changes, dumps the database, refreshes the official compose files while keeping your edits, restarts, and verifies health, migrations and port bindings. |

## Install

Copy a skill folder into your skills directory, named after the skill:

```bash
git clone https://github.com/JuanVqz/skills.git
mkdir -p ~/.claude/skills
cp -r skills/immich/upgrade ~/.claude/skills/immich-upgrade
```

Then ask Claude Code something like "upgrade Immich to the latest version". The skill also triggers on the "new version available" banner or a version number.

## Conventions

- Steps are numbered flat (Step 1, Step 2, ...), so it is easy to see which one was skipped.
- No secrets in output: skills read `.env` values they need without printing them.
- Nothing destructive without a backup and a check first.
