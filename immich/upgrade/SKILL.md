---
name: immich-upgrade
description: Upgrade a self-hosted Immich install (Docker Compose) to a newer release safely. Reads the release notes of every version in between for breaking changes, dumps the database first, refreshes the official compose files while keeping the user's own edits, bumps a pinned IMMICH_VERSION, restarts, and verifies health, migrations and port bindings. Use this whenever someone wants to update or upgrade Immich, sees the "new version available" banner in Immich, names a version like v3.4.0, asks whether an Immich release is safe to install, or wants to pull newer Immich images, even if they never say "upgrade".
---

# Upgrade Immich

Immich ships often, and it does not support downgrades: not even within a minor version, because every release can migrate the database. So an upgrade is a one-way door, and this skill walks through it in a fixed order: read first, back up, change on purpose, verify. Follow the steps in order and do not skip the backup or the checks to save time; they are the point.

Work on the machine that runs Immich (or over SSH to it). Ask the user for the host if it is not obvious.

## Step 1: Find the install

Locate the directory with Immich's `docker-compose.yml` and `.env`. Common places are the user's home (`~/immich-app`), `/opt`, or `/srv`; `docker compose ls` and `docker inspect immich_server --format '{{ index .Config.Labels "com.docker.compose.project.working_dir" }}'` usually reveal it.

Record, without printing any secret from `.env`:

- The running version: `curl -s localhost:2283/api/server/version` (or the published port from the compose file), or the image tag of `immich_server`.
- `IMMICH_VERSION` in `.env`. If it is `release` or a major-only tag like `v3`, the version can change on any `docker compose pull`. Recommend pinning an exact version (`v3.3.1`) as part of this upgrade, and explain why.
- `UPLOAD_LOCATION`, `DB_DATA_LOCATION`, `DB_USERNAME`, `DB_DATABASE_NAME` and the container names, for the later steps.
- The current port bindings: `docker compose ps --format '{{.Name}} {{.Ports}}'`. Step 8 compares against this.

## Step 2: Pick the target and read what changed

List recent releases and skip pre-releases (`-rc`):

```bash
gh release list -R immich-app/immich -L 10
```

Without `gh`, use `https://github.com/immich-app/immich/releases`.

Read the notes of every release between the running version and the target, not only the newest one. Breaking changes are often in an intermediate release. Look for: breaking changes, required manual steps, removed or renamed environment variables, Postgres or vector extension changes, and minimum mobile app versions. Also check the breaking-change label: `https://github.com/immich-app/immich/discussions?discussions_q=label%3Achangelog%3Abreaking-change`.

If the upgrade crosses a major version (v2 to v3), read Immich's migration post on `immich.app/blog` as well.

Then tell the user, in a few lines, what changes for them. If anything needs a manual step, a config change or a major version jump, stop and get their OK before going further. If nothing does, say so and continue.

## Step 3: Back up the database

Immich keeps daily database dumps in `UPLOAD_LOCATION/backups/`, but take a fresh one right before upgrading so it includes everything up to now:

```bash
docker exec immich_postgres pg_dump -U <DB_USERNAME> -d <DB_DATABASE_NAME> --clean --if-exists \
  | gzip > <safe-path>/immich-pre-<target>.sql.gz
gzip -t <safe-path>/immich-pre-<target>.sql.gz && ls -la <safe-path>/immich-pre-<target>.sql.gz
```

Write it somewhere other than the Postgres data directory, ideally on a different disk. Do not continue if the command fails, `gzip -t` fails, or the dump is much smaller than the newest one in `UPLOAD_LOCATION/backups/`. A dump that cannot be restored is not a backup.

## Step 4: Get the new release files

Download the release assets of both the running version and the target into a scratch directory:

```bash
for v in <current> <target>; do
  mkdir -p /tmp/immich-$v
  for f in docker-compose.yml example.env hwaccel.transcoding.yml hwaccel.ml.yml; do
    curl -sfL -o /tmp/immich-$v/$f https://github.com/immich-app/immich/releases/download/$v/$f
  done
done
diff -r /tmp/immich-<current> /tmp/immich-<target>
```

The second `diff` shows what upstream changed between the two releases: new image digests, new services, new variables. Often it is nothing or one digest.

## Step 5: Keep the user's edits on top of the new files

Most installs change the official `docker-compose.yml` a little: a bound port, hardware acceleration, an extra volume. Those edits must survive the upgrade, and nothing else should drift.

Find them by diffing the user's file against the official file of the **running** version:

```bash
diff /tmp/immich-<current>/docker-compose.yml docker-compose.yml
```

Then build the new file: start from the target's official `docker-compose.yml`, re-apply exactly those edits, and keep any comments the user wrote. Suggest marking each local edit with a short comment such as `# local: <why>`, so the next upgrade can find them at a glance.

Check the result: `diff /tmp/immich-<target>/docker-compose.yml docker-compose.yml` must show only the user's edits. If an upstream change touches a line the user edited, show both versions and ask.

Copy any `hwaccel.*.yml` file the compose file `extends` from the target release unmodified. Compare `example.env` of both releases and add new variables to the user's `.env` by hand, without printing existing values. Never touch `DB_PASSWORD`.

If the install is tracked in git, commit the new files before restarting, so the change is reviewable and reversible.

## Step 6: Bump the version

Set the target in `.env`:

```bash
sed -i "s/^IMMICH_VERSION=.*/IMMICH_VERSION=<target>/" .env
grep '^IMMICH_VERSION=' .env
```

On macOS, `sed -i ''`.

## Step 7: Pull and restart

```bash
docker compose pull
docker compose up -d
```

`up -d` recreates the containers whose image or configuration changed. A plain `restart` would keep the old image.

## Step 8: Verify

Wait until the containers report healthy, then check each of these, and fix or report anything that fails:

- `docker compose ps`: every service `healthy` (`immich_server`, `immich_machine_learning`, Postgres, Redis or Valkey).
- `curl -s localhost:2283/api/server/version` shows the target.
- `docker logs immich_server --since 10m 2>&1 | grep -iE "migrat|error"`: migrations end with "Finished running migrations", and no new errors appear.
- Port bindings match what Step 1 recorded. A careless merge can turn `127.0.0.1:2283:2283` back into `2283:2283`, which Docker publishes on every interface, past host firewalls like `ufw`.
- The web UI loads and shows recent photos.

## Step 9: Clean up and report

Remove the previous version's images once everything works:

```bash
docker image rm ghcr.io/immich-app/immich-server:<previous> ghcr.io/immich-app/immich-machine-learning:<previous>
```

Keep the pre-upgrade dump until the user has used the new version for a while.

Then give the user a short report: from and to versions, what the release notes changed for them, where the dump is, and anything unusual in the logs. Remind them to update the Immich mobile apps: the app and the server should be on the same major version.

## If it goes wrong

Do not downgrade the image against a migrated database. Instead:

1. Stop the stack: `docker compose down`.
2. Set `IMMICH_VERSION` back to the version the dump came from.
3. Restore the Step 3 dump into an empty database, following `https://docs.immich.app/administration/backup-and-restore` (the web "Restore from backup" flow on a fresh install, or the documented command-line restore).
4. Start the stack and verify as in Step 8.

Then read the release notes and Immich's GitHub issues for the error before trying again.
