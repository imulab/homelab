# vocab - deployment runbook

Vocabulary flash cards for Alfred, with FSRS spaced repetition and ElevenLabs
pronunciation cached in SQLite.

| | |
|---|---|
| Image | `ghcr.io/imulab/alfredapps-vocab` (private) |
| Tags | `sha-<short sha>` per build, `latest` on main |
| Source | `~/workspace/alfredapps/apps/vocab`, published by `.github/workflows/vocab-image.yml` |
| Port | `3001` container-internal only, reached through Caddy |
| Hostname | `vocab.absurdlab.dev` |
| Storage | one dataset bind mount at `/data` (SQLite database plus WAL) |
| Identity | uid `100`, gid `101`, non-root |
| Healthcheck | shipped in the image, `wget` against `/` every 30 s |

The runtime is a single compiled Bun binary on Alpine, 173 MB. There is no
`node_modules` and no build step on the NAS: Dockhand only pulls the image.

## Prerequisites

1. **Registry credentials.** GHCR is private. Create a personal access token
   (classic) with only the `read:packages` scope, then register `ghcr.io` in
   Dockhand with username `imulab` and that token as the password.
2. **Dataset.** `zfs create puddle/apps/vocab` and
   `chown -R 100:101 /mnt/puddle/apps/vocab`. The database and its WAL files are
   created on first start.
3. **DNS.** Add `vocab.absurdlab.dev` to the photonicat dnsmasq host entries
   pointing at the Caddy box (`192.168.20.11`), then refresh Passwall's DNS as
   described in `truenas/apps/README.md`. Caddy's wildcard certificate already
   covers the name.

## Deploy

1. In Dockhand create stack `vocab`, composing `truenas/apps/vocab/compose.yaml`.
2. Fill the stack environment panel from `.env.example`. Every key except
   `ELEVENLABS_API_KEY` is non-secret; leave their key-icon toggles off. Toggle
   the key icon ON for `ELEVENLABS_API_KEY` and paste the real key.
3. Deploy. The first pull needs the registry credentials from step 1.

## Verify

```sh
# On the NAS, the image healthcheck should report healthy
docker ps --filter name=vocab --format '{{.Status}}'

# Container-internal HTTP, no host port is published
docker exec vocab wget -qO- http://127.0.0.1:3001/ | head -c 200

# Through Caddy from the LAN
curl -I https://vocab.absurdlab.dev
```

Then open the app, add a word, and confirm the Parent page shows `ready` audio.
Word creation calls `api.elevenlabs.io`, so the container needs outbound
internet. Reviewing an existing word does not: the audio is already in SQLite.

## Updating

1. Push to `main` in `alfredapps`. The workflow publishes
   `ghcr.io/imulab/alfredapps-vocab:sha-<new sha>`.
2. Set `VOCAB_IMAGE_TAG` in Dockhand to the new tag.
3. Deploy. Dockhand pulls and recreates the container; the database is untouched.

## Rollback

Set `VOCAB_IMAGE_TAG` back to the previous `sha-` tag and deploy. Schema
migrations are forward-only: rolling back across a schema change can fail or
lose new columns. Check whether the release added a migration before rolling
back, and restore the dataset snapshot if it did.

## Backup

The database lives entirely under `VOCAB_DATA_PATH`, including the cached audio
blobs, so one snapshot covers both words and pronunciation. Take a ZFS snapshot
(for example `zfs snapshot puddle/apps/vocab@manual-$(date +%Y%m%d)`) and
replicate it with the rest of `puddle`. For a file-level copy, stop the container
first so the WAL is checkpointed, then copy `vocab.db`.

## Caveats

- **Secret redeploy (homelab decision #8).** After a daemon-initiated restart
  the container returns without `ELEVENLABS_API_KEY`; redeploy the stack in
  Dockhand to re-inject it. Words added meanwhile still save, with a pending
  pronunciation that the Parent page can generate later.
- **Free-tier voice rule.** Library voices such as Rachel return HTTP 402 on the
  free API plan. Alice, Lily, Matilda, Sarah, George, Brian and Daniel work.
- **Free-tier quota.** 10,000 characters per month, about 1,000 new words. The
  app needs `elevenlabs.io` attribution, which is in the app footer.
- **Access.** LAN-only through Caddy, no public DNS record. There is no app-level
  authentication, matching the Immich reachability decision; the surface is the
  household LAN.
- **Platform.** The image is `linux/amd64`, built for this NAS. An arm64 host
  needs a workflow change.
