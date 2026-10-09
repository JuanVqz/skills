# Skills

[Claude Code](https://claude.com/claude-code) skills I use for my own homelab, written to work on anyone's setup.

| Skill | What it does |
|---|---|
| [`immich/skills/upgrade`](immich/skills/upgrade/SKILL.md) (`immich-upgrade`, in the `immich-tools` plugin) | Upgrades a self-hosted Immich (Docker Compose): reads every release in between for breaking changes, dumps the database, refreshes the official compose files while keeping your edits, restarts, and verifies health, migrations and port bindings. |

## Install

As a plugin, from this repo's marketplace:

```text
/plugin install immich-tools --marketplace JuanVqz/skills
```

Or copy a single skill into your skills directory:

```bash
git clone https://github.com/JuanVqz/skills.git
mkdir -p ~/.claude/skills
cp -r skills/immich/skills/upgrade ~/.claude/skills/immich-upgrade
```

Then ask Claude Code something like "upgrade Immich to the latest version". The skill also triggers on the "new version available" banner, a version number, or an old install that needs to catch up.

The Immich skills are unofficial and not affiliated with Immich or FUTO. They read release notes and download release files from GitHub (`github.com/immich-app/immich`); nothing else leaves your machine.

## Conventions

- Steps are numbered flat (Step 1, Step 2, ...), so it is easy to see which one was skipped.
- No secrets in output: skills read `.env` values they need without printing them.
- Nothing destructive without a backup and a check first.

## Contributing

Contributions are welcome. If a skill misses a case on your setup (a different compose layout, a release with a new breaking change, a check that would have caught a problem), open an issue or a pull request. Keep the steps numbered flat and explain why a step exists, not only what it does.

## License

[MIT](LICENSE)
