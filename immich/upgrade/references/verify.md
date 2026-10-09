# Deep verification

The basic checks in SKILL.md prove the stack is up. These prove the library survived: the same assets are there, their thumbnails and videos still load, and anything the new version expects in the database exists. Run the "before" part during the backup step and the "after" part once the upgrade is healthy.

All queries run inside the Postgres container. Pipe SQL in through stdin instead of quoting it inside `ssh '...'`, because single quotes in the SQL break the outer quoting:

```bash
cat query.sql | docker exec -i immich_postgres psql -U <DB_USERNAME> -d <DB_DATABASE_NAME> -At
```

Table and column names below match Immich v3. Older versions used other names (for example `assets` instead of `asset`); if a query fails on the old version, list the tables with `\dt` and adapt rather than skipping the check.

## 1. Snapshot before the upgrade

Save the output to a file next to the dump, so the comparison after is exact:

```sql
-- assets by type, excluding the trash
select type, count(*) from asset where "deletedAt" is null group by 1 order by 1;
-- assets in the trash (Immich empties it on its own after 30 days by default)
select count(*) from asset where "deletedAt" is not null;
-- people, albums, users
select count(*) from person;
select count(*) from album;
select count(*) from "user";
-- generated files Immich tracks: thumbnails, previews, encoded videos
select type, count(*) from asset_file group by 1 order by 1;
```

## 2. Counts after the upgrade

Run the same queries. Live asset counts, people, albums and users should match. Explain any difference before calling the upgrade done:

- Fewer trashed assets is normal if the nightly trash cleanup ran in between.
- More assets is normal if phones uploaded during the upgrade.
- Fewer live assets, people or albums is not normal. Stop and investigate before deleting the backup.

## 3. Generated files exist on disk

A row in `asset_file` says a thumbnail exists; it does not prove the file is there. Sample a few hundred and check the files from inside the server container:

```sql
select path from asset_file where type = 'thumbnail' order by random() limit 300;
```

```bash
... | while read p; do docker exec immich_server test -f "$p" || echo "missing $p"; done | wc -l
```

Zero missing is the goal. If many are missing, the "Missing" job will not fix them: it only picks assets with **no** `asset_file` row, so rows pointing at absent files are skipped. Run **Generate Thumbnails > All** (Administration > Job Queues) instead. The same applies to encoded videos and **Transcode Videos > All**.

## 4. Data the new version expects

Read the release notes for new tables or columns that a migration does not backfill and that are filled by a job instead. Example from Immich v3: video codec and format moved to an `asset_video` table. A database restored or migrated from v2 had 0 rows there, every video thumbnail failed with `Missing video metadata for asset <id>`, and transcoding silently did nothing. The fix was **Extract Metadata > All**, then thumbnails and transcoding.

A generic check for this class of problem, after any upgrade:

```sql
select (select count(*) from asset where type = 'VIDEO' and "deletedAt" is null) as videos,
       (select count(*) from asset_video) as videos_with_metadata;
```

`videos_with_metadata` should be at least close to `videos`; it can be higher, because trashed and hidden videos (the motion part of Live Photos) also have rows. A count near zero is the problem. Apply the same idea to any new table the release notes mention.

## 5. Logs after the first jobs

Let the job queues run for a few minutes after the restart, then:

```bash
docker logs immich_server --since 15m 2>&1 | grep -E "ERROR|ENOENT|Missing video metadata" | sort | uniq -c | sort -rn | head
```

- `ENOENT ... profile/...` means a user's profile picture file is gone. Harmless; the user can upload a new one.
- `ENOENT ... thumbs/...` or `... encoded-video/...` points back to check 3.
- `Missing video metadata` points back to check 4.

## 6. The app works for a person

Open the web UI as a regular user: the timeline shows thumbnails, a video plays, search returns results, and a face in People has its picture. Ask the user to open the mobile app and confirm backup still runs.
