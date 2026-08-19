# Immich storage and Postgres briefing

> Resolves wayfinder ticket: Immich on-disk footprint + Postgres operational reality. Compiled 2026-08-19 via `/research` probe.
> Current at time of writing: **Immich v3.1.0** (published 2026-07-29, GitHub releases API). Docs live at **docs.immich.app** (immich.app/docs 307-redirects there; docs are versioned per release). All doc claims below were read from the v3-era docs on 2026-08-19.

## Storage anatomy: what the stack writes

From the official `docker/docker-compose.yml` at tag v3.1.0 (4 services: `immich-server`, `immich-machine-learning`, `redis`, `database`):

| Location | Lives in | Contents | Regenerable |
|---|---|---|---|
| `UPLOAD_LOCATION` → `/data` | bind mount (host path) | everything below | partially |
| `/data/library` | 〃 | **originals** after upload, laid out by storage template (default `Year/Year-Month-Day/Filename.ext`) | **no, irreplaceable** |
| `/data/upload` | 〃 | staging for in-app/CLI uploads before the move to `library` | mostly |
| `/data/thumbs` | 〃 | thumbhash + WebP preview + JPEG thumbnail + person/face thumbnails; small timeline thumbs and larger previews are mixed and **cannot be separated** | **yes** |
| `/data/encoded-video` | 〃 | pre-generated transcodes for browser/stream playback | **yes** |
| `/data/profile` | 〃 | profile pictures etc., "typically under 1 MB" | yes (annoying) |
| `/data/backups` | 〃 | built-in `pg_dump` output (daily 02:00, keep 14 by default) | see backup section |
| `model-cache` named volume → `/cache` | docker volume | ML models downloaded on demand, persisted so they are not re-fetched on restart | yes (re-download) |
| `DB_DATA_LOCATION` → `/var/lib/postgresql/data` | bind mount | **Postgres data dir** (metadata: users, albums, faces, tags, sidecar state, paths) | **no, irreplaceable** |
| Redis | **no volume at all** | Valkey 9 is used for BullMQ **job queues only**; no persistent state is stored there, so losing it drops queued jobs, not data (loss mode not explicitly documented) | n/a |

Notes:

- **Splitting is first-class.** `THUMB_LOCATION`, `ENCODED_VIDEO_LOCATION`, `PROFILE_LOCATION`, `BACKUP_LOCATION` env vars plus matching `/data/<name>` mounts move each category to its own device (official custom-locations guide). Caveats from the same guide: do not mount `upload/` and `library/` as separate bind mounts; storage metrics only track free space at `UPLOAD_LOCATION`, so other locations need separate monitoring. This maps cleanly onto separate ZFS datasets (regenerable vs irreplaceable).
- Container media root changed to `/data` in July 2025 ("change default media location to /data", 2025-07-29); older guides show `/usr/src/app/upload`.
- Postgres requirements (official): local **SSD**, never a network share; filesystem with user/group ownership (no NTFS/exFAT/WSL bind, use a docker volume there); `POSTGRES_INITDB_ARGS: --data-checksums` is set by the default compose (nice property for detecting torn pages after restores); if Docker RAM limits are used, give the DB ≥ 2 GB. `DB_STORAGE_TYPE: 'HDD'` exists if the DB is not on SSD.
- Reverse-geocoding datasets are fetched by the server outside `UPLOAD_LOCATION` (inside the container); exact location and size not verified this session.

**Sizing ratios:**

- **Doc-backed:** thumbnails + transcoded video add **"roughly 10-20%"** to library size on average; the database is typically **1-3 GB** (install/requirements page). Immich always keeps originals and generates transcodes alongside; transcodes are deletable and re-generatable via the Transcoding Policy setting.
- **Community-sourced real numbers:** thumbnails ~10-12.5% of a photos-only library (35 GB photos → 4.4 GB thumbs; "250 GB thumbs covers ~2.5 TB photos"); `encoded-video` is the wild card, one user at **65 GB** for a mixed library, another burn-in of ~1.7 TB on a very large import over 10 days (thumbs + previews + facial recognition). Video-heavy libraries push the ratio above the doc's 10-20%. Face thumbnails add extra per-person jobs.
- No built-in cap exists on thumb/encoded-video growth.

## External library (read-only import in place)

How it works (features/libraries + guides/external-library):

- Mount the existing directory into `immich-server` (`/srv/photos:/srv/photos:ro`), create an **External Library** owned by exactly one user, add one or more **import paths** (must be paths as the container sees them), then "Scan New Library Files". Files are **referenced in place, never copied or moved**; assets behave like normal ones (timeline, map, albums).
- Rescan triggers: a **nightly job scans all libraries** (cron configurable) plus manual scans; an **experimental watcher** (inotify) auto-imports new files but generally fails on network drives, needs `fs.inotify.max_user_watches` raised, and a hung watcher can block startup.
- Per-library glob **exclusion patterns** exist (`**/Raw/**`, `**/\@eaDir/**`, ...).

Constraints and semantics:

- **Sidecars:** with write access Immich writes XMP sidecars back into external folders on description/rating/tag edits. With a `:ro` mount sidecar writes fail; docs note edits can **silently fail with no warning** (issue #10538). Immich-only metadata (albums, descriptions if sidecar writes fail) lives in the DB and is **lost when a file is moved**, since rescan treats the new path as a new asset.
- **Deletion from the web UI** is only possible when the mount is **not** read-only; the `ro` flag disallows deleting external images in the UI (FAQ: with `ro`, trashed files even reappear after emptying the trash, since originals cannot be deleted). With `rw`, a web-UI delete removes the actual file on disk.
- **File deleted on disk** → next scan moves the asset to trash; 30 days later it is purged with its Immich metadata. Deleting the whole library deletes its assets immediately (background job, cleaned up by the nightly cron).
- **Duplicate detection does not apply** to external libraries (it is upload-library only, per-library), so duplicates can appear across import paths.
- No official album import for external libraries (community workarounds in discussion #4279).

**Fit for a Jellyfin-owned directory: yes, and it is the canonical pattern.** Mount the shared directory `:ro` into Immich; Jellyfin keeps its own read. Consequences of `:ro`: Immich can never delete or rename Jellyfin's media, no XMP sidecar writes into that tree (edits stay DB-only, and are lost if files move), and rescans pick up changes Jellyfin-side tooling makes. Community reports confirm both services pointing at the same path read-only works; keep Immich's own upload library separate from the external tree.

## Postgres and the vector extension: required versions and why they are pinned

**Current requirement (docs, postgres-standalone page):** Postgres **>= 14, < 20**, **VectorChord >= 0.3, < 2.0**, and pgvector **>= 0.7, < 0.9** (VectorChord builds on pgvector's types). Immich **checks at startup and refuses to start** on incompatible versions. pgvecto.rs support was **dropped in Immich 3.0**.

**Default compose image (v3.1.0):** `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`, digest-pinned. So: Postgres 14, VectorChord 0.4.3, and pgvecto.rs 0.2.0 still bundled purely so **pre-migration backups remain restorable** (a leaner `:14-vectorchord0.4.3` tag without pgvecto.rs exists). The image line covers other majors too (e.g. `16-...`, and `18-vectorchord0.5.3` is in active community use).

**History (release notes + compose commit log):**

- Through v1.132: `tensorchord/pgvecto-rs:pg14-v0.2.0` (pgvecto.rs, deprecated upstream; Immich stayed on 0.2.x because newer pgvecto.rs changed its SQL API).
- **v1.133.0 (2025-05-21), breaking:** switch to VectorChord via first-party image `ghcr.io/immich-app/postgres:14-vectorchord0.3.0-pgvectors0.2.0`. Automatic in-place DB migration at startup ("seconds to minutes"; large libraries sit on "Reindexing clip_index/face_index" for a while). Do not downgrade below 1.133.0 afterwards.
- VectorChord tracked 0.3.0 → 0.4.1 (2025-05-28) → 0.4.3 (2025-06-20); digest-only updates since (Oct 2025 → current digest).

**Why pinned:** smart search and facial recognition are stored as vector indexes (`clip_index`, `face_index`) inside the DB; their on-disk format is extension-version-specific. Extension updates require `ALTER EXTENSION vchord UPDATE;` plus `REINDEX INDEX clip_index; REINDEX INDEX face_index;`. Version drift breaks startup or the search indexes, and backups taken on VectorChord only restore into VectorChord-capable images.

## Upgrade hazards: Immich upgrades vs Postgres major bumps

- **Routine Immich upgrades** are `docker compose pull && up -d` after reading release notes; the DB image tag rarely moves. Routine upgrades do not mandate a manual dump, but Immich's **built-in automatic backups** (daily 02:00, keep 14, in `/data/backups`) institutionalize dump-before-risk anyway. Downgrades are unsupported, mobile apps must be upgraded before the server, and breaking changes are confined to major releases.
- **Postgres major bumps: data files are not portable.** Swapping the image across majors against the old volume fails with `FATAL: database files are incompatible with server`. Same-major image swaps (e.g. `17-vectorchord0.4.3-...` → `17-vectorchord0.5.3-...`) are reported to just work.
- **Blessed major-upgrade path is dump/restore via Immich's own UI** (community-documented in discussion #24831, no maintainer-authored page): upgrade Immich to ≥ **v2.5.5** (backup/restore fix), take a dump from the UI, stop `immich-server`, point the DB service at the new image tag **and a fresh empty volume** (note: PG 18 images move the mount to `/var/lib/postgresql`), start DB then server, restore via the web UI (single transaction, restore point, auto-rollback). Rollback = revert compose to old image + old volume; the old DB is untouched. Belt-and-braces pattern: `docker exec -t immich_postgres pg_dumpall --clean --if-exists --username=postgres | gzip > dump.sql.gz` first. `pg_upgrade` is not the shipped path for compose users.
- **How often it bites:** compose users rarely, because the default has been Postgres 14 since the pgvecto.rs era and Immich's own images keep major bumps opt-in. It bites platform-packaged installs: the TrueNAS app lineage moved PG 15 → 18 and the chart's internal `pgvecto_upgrade` (pg_upgrade-based) container crashed with "Old PostgreSQL [15] binaries not found", wedging the app until users hand-edited `ix_values.yaml` or nuked the DB and restored from the auto-dumps (truenas/apps issue #4628; several forum threads). Unraid users hit the same class of failure.
- Dumps are small relative to files (~300 MB dump for a 60 GB library), so dump-before-every-upgrade is cheap.

## Backup story

**Official guidance (backup-and-restore page):** exactly two things matter, and they must pair:

1. **Database** (in `DB_DATA_LOCATION`): "essential"; Immich never rescans the library folder to rebuild metadata.
2. **Filesystem**: originals are critical, i.e. `library`, `upload`, `profile`; `thumbs` and `encoded-video` are optional (skip them and re-run those jobs after restore). Back up any external library trees too, plus the stack config.

**Ordering and quiescing (doc-stated):** best practice is to **stop `immich-server`** during backup so nothing changes. If you cannot: **database first, filesystem second**; the worst case is then orphaned files (re-uploadable), whereas the reverse order risks the DB referencing missing files, i.e. broken assets. The built-in restore guarantees a consistent DB via a single transaction with an automatic restore point and rollback on failure. 3-2-1 is the stated strategy; Immich does not do filesystem backups itself.

**Built-in mechanism:** daily 02:00 `pg_dump` into `UPLOAD_LOCATION/backups`, keep 14; manual dump from Job Queues; restore from UI (or `.sql.gz` upload) onto a **fresh install**; the restore process changed in v2.5.0. Caveat: dumps land inside `UPLOAD_LOCATION`, so a files-only backup tool that includes the media root picks them up for free.

**Community patterns and consistency tradeoffs:**

- **ZFS snapshots of DB + files together:** Postgres's own documentation blesses "consistent snapshots" of a running server: restore replays WAL exactly like a crash, provided the snapshot is atomic and **WAL files are included** (run `CHECKPOINT` first to shorten recovery). OpenZFS guarantees `zfs snapshot -r` creates all child snapshots **at the same point in time**. So a single recursive snapshot of one parent dataset containing `pgData` + `data` yields a crash-consistent, mutually consistent DB+files pair. Important: this only holds if both live under one parent and the snapshot is recursive; separate per-dataset snapshot tasks scheduled at different times give no such guarantee.
- **Community experience (r/immich, forums.truenas.com):** no first-hand failures of zfs-snapshot restores reported; the debate is theoretical. Strong consensus on two caveats: never put the Postgres data dir on **NFS** (roundly rejected), and **snapshots are not backups** until replicated off-host. A popular pattern is timing the zfs snapshot right **after** the 02:00 built-in dump lands, so the snapshot contains both a WAL-recoverable DB and a clean `.sql.gz`. The most conservative users do `docker compose down` → snapshot → up, or Proxmox "stop" mode backups.
- **restic/app-level:** plain file-level backup of media + dumps works (this is what Dockhand's own backup plus Immich's built-in dumps would give you), with the DB-first/files-second ordering rule and no snapshot atomicity across the two streams. Restore drills reported painful across versions (e.g. restoring a 1.124 dump on 2.6.3 hits `DROP DATABASE cannot run inside a transaction block`); prefer matching versions when restoring.

**TrueNAS app specifics (docs/install/truenas):** the community-train app defaults to host-path datasets under a parent (e.g. `/mnt/tank/immich/{data,pgData}`; pgData owned by UID 999 `netdata`, data by `apps` 568, SMB ACL mode Passthrough if storage templates are used); default **ixVolumes are explicitly "Not recommended"** (harder to snapshot/replicate, data-loss risk on app delete). The docs even suggest per-folder datasets (`library`, `upload`, `thumbs`, `profile`, `encoded-video`, `backups`) for finer control, which is exactly the dataset-design question the follow-up ticket owns.

## Homelab implications (input to the dataset-design ticket, not a decision)

- One ZFS parent for everything Immich-owned (`pgData` + `data`, plus separate children for regenerable `thumbs`/`encoded-video`) preserves recursive-snapshot atomicity for DB+files, while child datasets still allow split quotas/record sizes and excluding regenerables from replication.
- The Jellyfin-shared media tree should be mounted `:ro` into Immich as an external library; Immich then adds no writes to it at all, so it needs no Immich-side dataset design beyond what Jellyfin already has.
- Immich's own `/data/backups` dumps plus the built-in scheduler cover app-level DB dumps; a recursive zfs snapshot taken after the 02:00 dump plus native replication covers crash-consistency and off-box copies; Dockhand's existing backup covers stack files and secrets. The open question for the next ticket is which mechanism is authoritative for what, and whether thumbs/encoded-video replicate at all.

## Confidence notes

- **Doc-backed (docs.immich.app, read 2026-08-19, v3-era):** storage locations and custom-location env vars; `/data` layout incl. `backups`, `profile`, `thumbs` being inseparable; built-in backup schedule/retention and restore semantics (incl. v2.5.0 process change, single-transaction restore); DB-first/files-second ordering; stop-server best practice; 10-20% overhead figure; DB 1-3 GB, SSD, no NFS/NTFS; Valkey queue role and absence of a volume; external library mechanics, `:ro` semantics (no UI deletion, no sidecar writes, silent-fail caveat), nightly scan, watcher limits, exclusion globs, duplicate-checking scope; Postgres >= 14 < 20, VectorChord >= 0.3 < 2.0, pgvector >= 0.7 < 0.9, startup version check, pgvecto.rs dropped in 3.0, extension update/reindex procedure; upgrade procedure and no-downgrade policy; TrueNAS app dataset guidance and ixVolume warning.
- **Doc-backed (GitHub, read 2026-08-19):** v3.1.0 latest release (2026-07-29); exact compose images/volumes at v3.1.0; v1.133.0 (2025-05-21) VectorChord migration notes incl. old/new images, migration behavior, downgrade warning; compose commit history for image/version timeline; Postgres file-system-snapshot guidance (postgresql.org current docs); OpenZFS `zfs snapshot -r` atomicity (zfs-snapshot.8).
- **Community-sourced:** thumbnail size data points (10-12.5% of photos, 65 GB encoded-video example, 1.7 TB import burn-in); Jellyfin + Immich sharing one read-only tree; discussion #24831 dump/restore major-upgrade recipe (user-authored, not maintainer docs) incl. PG18 mount-path change and successful 14→18 report; TrueNAS PG15→18 upgrade breakage and workarounds (truenas/apps #4628); r/immich zfs-snapshot thread verdict (no failures reported, NFS-hosted DB rejected, snapshot-after-dump pattern, compose-down-then-snapshot pattern); TrueNAS pg_dump/pgAdmin guide quirks (discussion #8809); cross-version restore failures (forum thread).
- **Undetermined:** whether any WAL-level guarantee beyond crash recovery matters for zfs restores at Immich's write volume (no authoritative Immich-specific statement; PG docs are the closest source); exact ML model-cache footprint (models on demand, persisted in `model-cache`; sizes not verified); reverse-geocoding dataset location/size; whether a maintainer-endorsed Postgres-major-upgrade page exists (none found; the recipe is community-maintained only).

## Sources

[docs.immich.app](https://docs.immich.app/) · [requirements](https://docs.immich.app/install/requirements) · [docker compose install](https://docs.immich.app/install/docker-compose) · [backup and restore](https://docs.immich.app/administration/backup-and-restore) · [custom locations](https://docs.immich.app/guides/custom-locations) · [external libraries](https://docs.immich.app/features/libraries) · [external library guide](https://docs.immich.app/guides/external-library) · [XMP sidecars](https://docs.immich.app/features/xmp-sidecars) · [storage template](https://docs.immich.app/administration/storage-template) · [FAQ](https://docs.immich.app/FAQ) · [upgrading](https://docs.immich.app/install/upgrading) · [standalone Postgres](https://docs.immich.app/administration/postgres-standalone) · [architecture](https://docs.immich.app/developer/architecture) · [TrueNAS install](https://docs.immich.app/install/truenas) · [compose @ v3.1.0](https://github.com/immich-app/immich/blob/v3.1.0/docker/docker-compose.yml) · [compose commit history](https://github.com/immich-app/immich/commits/main/docker/docker-compose.yml) · [v1.133.0 release notes](https://github.com/immich-app/immich/releases/tag/v1.133.0) · [DB upgrade discussion #24831](https://github.com/immich-app/immich/discussions/24831) · [TrueNAS DB backup guide #8809](https://github.com/immich-app/immich/discussions/8809) · [TrueNAS: Immich fails PG18 update](https://forums.truenas.com/t/immich-app-fails-postgres-18-update/64637) · [TrueNAS: restore from 1.124 on 2.6.3](https://forums.truenas.com/t/immich-2-6-3-restore-backup-from-1-124-2/64936) · [r/immich: DB backups via zfs snapshots](https://www.reddit.com/r/immich/comments/1uzrhvx/database_backups_via_zfs_snapshots/) · [r/immich: upgrading PG 14 to 18](https://www.reddit.com/r/immich/comments/1s0sfdv/upgrading_postgresql_14_to_18/) · [r/immich: thumbnail sizes](https://www.reddit.com/r/immich/comments/198t00d/how_much_space_does_thumbnail_take/) · [r/immich: encoded-video 65GB](https://www.reddit.com/r/immich/comments/1l3c9ja/exccessive_storage_use_for_generated_thumbnails/) · [PostgreSQL docs §25.2 file-system backups](https://www.postgresql.org/docs/current/backup-file.html) · [OpenZFS zfs-snapshot(8)](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-snapshot.8.html)
