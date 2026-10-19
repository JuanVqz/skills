# Skills

[Claude Code](https://claude.com/claude-code) skills I use for my own homelab, written to work on anyone's setup.

| Skill | What it does |
|---|---|
| [`immich/skills/upgrade`](immich/skills/upgrade/SKILL.md) (`immich-upgrade`, in the `immich-tools` plugin) | Upgrades a self-hosted Immich (Docker Compose): reads every release in between for breaking changes, dumps the database, refreshes the official compose files while keeping your edits, restarts, and verifies health, migrations and port bindings. |
| [`bookorbit/skills/upgrade`](bookorbit/skills/upgrade/SKILL.md) (`bookorbit-upgrade`, in the `bookorbit-tools` plugin) | Upgrades a self-hosted BookOrbit (Docker Compose): backs up the database and app data, reads every release in between, refreshes the official compose file while keeping your edits, pins the image instead of `latest`, and verifies health, migrations, port bindings and the library counts. |

## Install

As a plugin, from this repo's marketplace:

```text
/plugin install immich-tools --marketplace JuanVqz/skills
/plugin install bookorbit-tools --marketplace JuanVqz/skills
```

With the [skills CLI](https://skills.sh/juanvqz/skills), for Claude Code or other agents:

```bash
npx skills add JuanVqz/skills --skill immich-upgrade
npx skills add JuanVqz/skills --skill bookorbit-upgrade
```

Or copy a single skill into your skills directory:

```bash
git clone https://github.com/JuanVqz/skills.git
mkdir -p ~/.claude/skills
cp -r skills/immich/skills/upgrade ~/.claude/skills/immich-upgrade
```

Then ask Claude Code something like "upgrade Immich to the latest version" or "upgrade BookOrbit". The skill also triggers on the "new version available" banner, a version number, or an old install that needs to catch up.

The Immich skills are unofficial and not affiliated with Immich or FUTO, and the BookOrbit skills are unofficial and not affiliated with BookOrbit. They read release notes and download release files from GitHub (`github.com/immich-app/immich`, `github.com/bookorbit/bookorbit`); nothing else leaves your machine.

## Conventions

- Steps are numbered flat (Step 1, Step 2, ...), so it is easy to see which one was skipped.
- No secrets in output: skills read `.env` values they need without printing them.
- Nothing destructive without a backup and a check first.

## Contributing

Contributions are welcome. If a skill misses a case on your setup (a different compose layout, a release with a new breaking change, a check that would have caught a problem), open an issue or a pull request. Keep the steps numbered flat and explain why a step exists, not only what it does.

## License

[MIT](LICENSE)
