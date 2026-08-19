# Immich deployment and reverse-proxy briefing

> Resolves wayfinder ticket: what a correct Immich compose deployment looks like and what its reverse-proxy integration requires. Compiled 2026-08-19 via `/research` probe against primary sources: the official compose/env **release assets** (downloaded from `releases/latest`, resolving to v3.1.0), GHCR image configs (v3.1.0, read via registry API), the Immich source tree (sparse clone of `main`, commit `8aa95c6`, same as the v3.1.0 build), and docs.immich.app pages (reverse-proxy page last updated 2026-07-27). All doc claims were read on 2026-08-19.
> Companion briefs: [immich-storage-and-postgres.md](./immich-storage-and-postgres.md) (volume anatomy, sizing, Postgres care), [immich-hardware-acceleration.md](./immich-hardware-acceleration.md) (OpenVINO/QSV, CPU sizing).

## TL;DR

- **Current stable: v3.1.0, published 2026-07-29.** The official compose ships **4 services**: `immich-server`, `immich-machine-learning`, `redis` (Valkey 9, digest-pinned), `database` (Immich's own VectorChord Postgres image, digest-pinned). `immich-web` and the `immich-proxy` nginx container are gone: dropped from the compose at v1.92.0 (June 2024), and the images are no longer published at all (all GHCR tags 404, checked 2026-08-19).
- **Direct Caddy-to-server proxying is the officially documented pattern.** The reverse-proxy docs list Nginx, Caddy, Apache, Traefik as supported, and the official Caddy example is a bare `reverse_proxy http://<host>:2283`. Everything nginx needs special directives for (upload size, request buffering, 600s timeouts, websocket upgrade headers) Caddy does by default: no body-size limit, no buffering, no proxy read/write timeouts, native websocket tunneling (Caddy docs, viewed 2026-08-19).
- **Only one port matters: 2283 on `immich-server`** (API + web UI + microservices in one container). ML (3003), Valkey (6379), Postgres (5432) are internal to the compose network and need no publishing.
- **Healthchecks ship in the images**, which fits our "inherit image-shipped healthchecks" rule exactly: server pings `/api/server/ping` and expects `{"res":"pong"}`, ML pings `/ping`, Postgres runs `healthcheck.sh`, and the compose adds `redis-cli ping` for Valkey inline. The compose file's `healthcheck: disable: false` entries just keep them enabled.
- **Secrets surface is tiny: `DB_PASSWORD` only.** `JWT_SECRET` no longer exists anywhere (docs or source). `IMMICH_MACHINE_LEARNING_URL` is no longer in the example env; the server defaults it to `http://immich-machine-learning:3003`, matching the compose service name.
- **Pin exact versions.** Docs endorse either the `:v3` major metatag or an exact pin like `v3.1.0`; semver with breaking changes confined to majors, no backports, no downgrades. Upgrade is `docker compose pull && docker compose up -d` after bumping `IMMICH_VERSION`. For Dockhand manual deploys, pin exact and upgrade deliberately.

## 1. Service topology (official compose, v3.1.0)

The official compose is a **release asset**, not a path in the repo you should copy: `https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml` (the repo's `docker/README.md` warns the file on `main` may not match the latest release). Fetched 2026-08-19, it contains:

| Service | Image (as shipped) | Version tag | Ports | Volumes |
|---|---|---|---|---|
| `immich-server` (container `immich_server`) | `ghcr.io/immich-app/immich-server` | `${IMMICH_VERSION:-release}` | **2283/tcp, published `2283:2283`** (only published port in the file) | `${UPLOAD_LOCATION}:/data`, `/etc/localtime:ro` |
| `immich-machine-learning` (`immich_machine_learning`) | `ghcr.io/immich-app/immich-machine-learning` | `${IMMICH_VERSION:-release}` (same var; add `-openvino` etc. suffix for ML acceleration) | none published; listens on 3003 internally | named volume `model-cache:/cache` |
| `redis` (`immich_redis`) | `docker.io/valkey/valkey:9` | **digest-pinned** in the compose (`@sha256:8e8d64b4...`) | none published; 6379 internal | none (queue state only) |
| `database` (`immich_postgres`) | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` | **digest-pinned** (`@sha256:bcf63357...`) | none published; 5432 internal | `${DB_DATA_LOCATION}:/var/lib/postgresql/data`, `shm_size: 128mb` |

Shared version tag: **only `immich-server` and `immich-machine-learning` move together** via `IMMICH_VERSION`. Postgres and Valkey are pinned independently (Immich rotates their digests between its own releases; e.g. the Valkey digest differs between v2.4.0, v3.0.0 and v3.1.0 assets). All four services use `restart: always`; the server has plain `depends_on: [redis, database]` with no health conditions. Both app services get `env_file: .env`. Hardware acceleration is opt-in via commented `extends:` blocks referencing `hwaccel.transcoding.yml` / `hwaccel.ml.yml` (covered in the hardware brief).

How it got here (verified from historical release assets):

- **v1.92.0 (June 2024)**: web UI merged into `immich-server` (compose maps host `2283` to the server's old internal 3001), `immich-proxy` gone from the compose, `immich-microservices` still a separate service.
- **By v2.0.0 (2026-07-02)**: microservices also merged into the server; the four-service shape above is unchanged through v3.1.0.
- **The old images are unpublished**: `ghcr.io/immich-app/immich-proxy` and `immich-web` return 404 for every tag checked, including historical ones (v1.161.0, v2.0.0, v3.0.0). Anyone still running `immich-proxy` from an old cached image is on their own; nothing in current docs mentions it.

Also in the repo's `docker/` directory (not in the release asset): `docker-compose.rootless.yml` (rootless variant) and `docker-compose.prod.yml` (a build/dev-oriented file). The install docs use neither for the standard path.

## 2. Terminating at Caddy directly

**Yes, officially supported, and it is the intended architecture now.** The current reverse-proxy docs (docs.immich.app/administration/reverse-proxy, last updated 2026-07-27, viewed 2026-08-19) never mention Immich's own proxy container; every example fronts **`immich-server` on port 2283** with an external proxy: Nginx, **Caddy**, Apache, and Traefik.

### What the docs require of any proxy

1. **Forward all headers and set** `Host`, `X-Real-IP`, `X-Forwarded-Proto`, `X-Forwarded-For` to correct values.
2. **Allow big enough uploads** (the nginx example uses `client_max_body_size 50000M;` plus `proxy_request_buffering off;`, noting buffering off "prevents OOM on the reverse proxy server and makes uploads twice as fast").
3. **Long-enough timeouts** (nginx example: `proxy_read_timeout 600s` etc.; the Traefik example raises entrypoint `respondingTimeouts` because the 60s default makes "videos stop uploading after 1 minute (Error Code 499)").
4. **Websocket upgrade support** (nginx needs explicit `proxy_http_version 1.1` + `Upgrade`/`Connection "upgrade"` headers).
5. **Serve Immich at the root of a (sub)domain**: sub-path serving is explicitly unsupported.
6. If using Let's Encrypt http-01, make sure `/.well-known/immich` actually routes to Immich, or the mobile app "may run into connection issues".

### The official Caddy example, verbatim

```
immich.example.org {
    reverse_proxy http://<snip>:2283
}
```

That is the entire Caddy section: no size limits, no timeouts, no websocket directives. This is consistent with Caddy's documented defaults (caddyserver.com/docs, viewed 2026-08-19):

- **WebSockets**: "The proxy also supports WebSocket connections, performing the HTTP upgrade request then transitioning the connection to a bidirectional tunnel." No configuration needed. (One caveat: by default Caddy forcibly closes websocket connections on config reload; `stream_close_delay` can soften that.)
- **Request body**: no default max body size; request/response buffering is opt-in (`request_buffers`/`response_buffers`, both discouraged). So the "big enough uploads" requirement is satisfied by doing nothing.
- **Timeouts**: `response_header_timeout`, `read_timeout`, `write_timeout`, `stream_timeout` all default to **no timeout** (only `dial_timeout` defaults to 3s). Nothing to raise.

So the nginx/Apache/Traefik directive gymnastics are proxy-specific workarounds; with Caddy the requirements are met by the default `reverse_proxy`. The docs' Traefik example is also instructive for us because it is **label-driven on the compose service** (`traefik.http.services.immich.loadbalancer.server.port: 2283`, plus a note that the proxy must share a network with `immich-server`), which is architecturally identical to our `caddy.reverse_proxy "{{upstreams 2283}}"` label pattern on the shared external docker network.

### Header details worth knowing

- Caddy's `reverse_proxy` sets `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host` automatically and passes `Host` through; it does **not** set `X-Real-IP` by default. The official Caddy example doesn't add it either, and Caddy+Immich is the blessed path, so `X-Real-IP` is evidently not load-bearing (adding `header_up X-Real-IP {remote_host}` costs nothing if we want parity with the docs' header list).
- Whether Immich *uses* the forwarded client IP is gated by Express `trust proxy`. Source (server/src/app.common.ts): `app.set('trust proxy', ['loopback', ...network.trustedProxies])`, where `IMMICH_TRUSTED_PROXIES` defaults to `['linklocal', 'uniquelocal']` (server/src/repositories/config.repository.ts; docs call it a "List of comma-separated IPs set as trusted proxies"). Those presets cover loopback, 169.254/16 + fe80::/10, and IPv6 ULA, but **not RFC1918 IPv4** like a Docker bridge network (172.16/12). Practical effect for us: with Caddy as a container peer, Immich will attribute requests to the Caddy container IP until we set e.g. `IMMICH_TRUSTED_PROXIES=172.16.0.0/12` on the server. That affects logging/IP-derived niceties, not auth or uploads.

### Community guides (consistent with the above)

- Immich discussion #7164: dockerized Caddy + Immich; key point is the docs' `http://<ip>:2283` form only applies when Caddy can reach the IP; on a shared docker network use `immich-server:2283` (service name).
- Issue #11666: reverse proxy only works on a subdomain, reinforcing the no-sub-path rule.
- r/immich thread `1cdeg47`: what address belongs in `reverse_proxy` (same answer: container name on a shared network, or host IP with the port published).

### Homelab mapping (derived, not official)

Drop `ports: '2283:2283'` entirely (nothing else needs host exposure), attach the external Caddy network to `immich-server`, and label it with upstream port 2283. That reproduces the docs' Traefik-label architecture with our Caddy edge. The internal healthcheck is unaffected (it curls `localhost:2283` inside the container). `2283` is also worth a host publish *only* if we ever want to bypass Caddy for LAN access; the compose default publishes it.

## 3. Env and config surface

The env file ships as another release asset: `https://github.com/immich-app/immich/releases/latest/download/example.env` (the install guide's `wget -O .env ...example.env`). The complete v3.1.0 example, verbatim minus comments:

```bash
UPLOAD_LOCATION=./library      # host path for /data
DB_DATA_LOCATION=./postgres    # host path for Postgres; "Network shares are not supported"
# TZ=Etc/UTC                   # uncomment + set
IMMICH_VERSION=v3              # "You can pin this to a specific version like v2.1.0"
DB_PASSWORD=postgres           # "change it to a random password", chars A-Za-z0-9 only
DB_USERNAME=postgres           # below "do not need to be changed"
DB_DATABASE_NAME=immich
```

That is the **entire** required surface. Notably absent versus older guides:

- **`JWT_SECRET` is gone.** Zero matches in the current env docs or the server source (sessions/auth no longer need it). Old tutorials that set it are outdated.
- **`IMMICH_MACHINE_LEARNING_URL` is no longer needed in the env file.** The server defaults it to `http://immich-machine-learning:3003` (server/src/config.ts), which resolves to the compose service name. It still exists as an override (e.g. point at a remote ML host, or multiple URLs), and `IMMICH_MACHINE_LEARNING_ENABLED=false` disables ML entirely. ML listens on `IMMICH_PORT` default 3003 (its own env namespace).

Other documented variables relevant to deployment mechanics (env docs, viewed 2026-08-19):

- **Database**: `DB_URL` (full connection string; "When DB_URL is defined, the DB_HOSTNAME, DB_PORT, DB_USERNAME, DB_PASSWORD and DB_DATABASE_NAME database variables are ignored"), `DB_HOSTNAME` (default `database`), `DB_PORT` (5432), `DB_SSL_MODE`, `DB_VECTOR_EXTENSION` (auto between vectorchord/pgvector), `DB_STORAGE_TYPE` (SSD/HDD tuning). All `DB_*` vars must reach all workers.
- **Redis**: `REDIS_HOSTNAME` (default `redis`, which is the compose service name even though the image is Valkey), `REDIS_PORT` (6379), `REDIS_USERNAME`/`REDIS_PASSWORD`/`REDIS_DBINDEX`, or `REDIS_URL` (odd format: "must start with ioredis:// and then include a base64 encoded JSON string"). No password by default in the compose; it stays network-local.
- **Listener**: `IMMICH_HOST` (0.0.0.0), `IMMICH_PORT` (2283 for the server, 3003 for ML). The server healthcheck honors both.
- `IMMICH_TRUSTED_PROXIES` (see section 2), `IMMICH_ALLOW_SETUP` (set `false` after onboarding to disable `/auth/admin-sign-up`), `TZ`.
- **Env changes require recreation, not restart**: "To change environment variables, you must recreate the Immich containers" (docs). With Dockhand that is a redeploy, which we do anyway.
- Docker-secrets `_FILE` variants exist for the `DB_*` and `REDIS_PASSWORD` variables; irrelevant here since this homelab has no Compose/daemon secrets support.

**Secrets classification**: `DB_PASSWORD` is the only true secret in the default deployment (docs: change it; it is local-auth only since Postgres is not exposed). Everything else in the env file is non-secret configuration. The Dockhand caveat applies to it directly: if the daemon restarts the containers out-of-band, the injected `DB_PASSWORD` is lost and the server cannot authenticate to Postgres until a manual redeploy. Mitigations within our model: treat a daemon restart followed by a dead Immich as "redeploy the stack", same as the Caddy token case.

## 4. Version and upgrade model

- **Current stable: v3.1.0** (published 2026-07-29; confirmed via GitHub releases API on 2026-08-19; latest is not a prerelease, and no v3.1.x patch has shipped since).
- **Observed cadence** (releases API, dates seen): monthly-ish minors with prompt patches, majors preceded by public RCs: v2.6.3 2026-03-26, v2.7.0..v2.7.5 2026-04-07..04-13, v3.0.0-rc.0..rc.3 2026-06-15..06-26, v3.0.0 2026-07-02, v3.0.1/02/03 07-02/07-09/07-15, v3.1.0 07-29. No official written cadence statement exists in current docs (older "every other week" claims are stale).
- **Tag policy** (upgrading docs, "Versioning Policy"): semver; "We intend for breaking changes, including those to the API or deployment, to be limited to major version releases". Options: the `release` tag (compose fallback; latest stable), a **major metatag** like `:v3` ("These metatags do not follow release candidates"), or an exact pin. "We do not backport patches... we encourage all users to run the most recent stable release... Downgrading to an earlier version, even within the same minor version, is not supported." Mobile apps work with current + prior major; "the server is only compatible with the matching major version", so upgrade mobile clients first. For a manually-deployed homelab stack, an exact pin (`v3.1.0`) read at deploy time is the most controlled; the `v3` metatag is the low-touch alternative.
- **Upgrade procedure** (docs, verbatim core): read the release notes (breaking changes are tracked under the `changelog:breaking-change` discussion label), update `IMMICH_VERSION` if pinned, then `docker compose pull && docker compose up -d`, then `docker image prune`. Nothing else.
- **Database migrations run automatically on startup**: "Immich will make some changes to the DB during startup, which can take seconds to minutes to finish, depending on hardware and library size"; logs can look stuck at `Reindexing clip_index` / `Reindexing face_index` past ~100k assets; if no errors, wait. Docs advise backing up the database before upgrade work (Immich also takes daily `pg_dump` backups into `/data/backups` by default; see the storage brief).
- **VectorChord migration is a non-issue for a fresh install**: current compose already ships `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`. Only pre-v1.133.0 pgvecto.rs installs need the documented migration, after which "you should not downgrade Immich below 1.133.0".
- **v3.0.0 breaking changes**: per the release notes, mostly API-endpoint changes affecting third-party integrations; "For the vast majority of users, updating works exactly as it always has" (change `IMMICH_VERSION` to `v3`, pull, up). Full migration guide: immich.app/blog/v3-migration.
- **ML model cache** persists in the `model-cache` named volume, so upgrades do not re-download models (regenerable if lost; see storage brief).
- **Admin CLI is `immich-admin`** (not "immich-server-admin"; that name does not exist). Preinstalled in the server image; docs (administration/server-commands) say to connect to the `immich_server` container and run e.g. `immich-admin reset-admin-password`. Full set: `reset-admin-password`, `disable/enable-password-login`, `disable/enable-oauth-login`, `enable/disable-maintenance-mode` (prints a tokenized login URL), `list-users`, `grant-admin`, `revoke-admin`, `version`, `change-media-location`, `schema-check` (verify migrations / schema drift). Useful break-glass surface for a headless deploy.
- **Release candidates** are opt-in: Admin settings > Version check can switch "Stable" to "Release candidate". Watchtower-style auto-updates: no guidance in current docs beyond a stale link reference; nothing recommends auto-updating (undetermined stance, effectively "manual").

## 5. Healthchecks (all four services, image-shipped)

This is the strongest fit with our stack convention: **the images define HEALTHCHECKs and the compose simply keeps them** (`healthcheck: disable: false` on server, ML, and database; the Valkey service declares its test inline). The VectorChord FAQ in the upgrade docs states this explicitly: when the old inline compose healthcheck was removed, "These lines are now incorporated into the image itself along with some additional tuning."

Verified from the v3.1.0 image configs (GHCR registry API) and the scripts in the repo:

| Service | Healthcheck (image-level) | What it does |
|---|---|---|
| `immich-server` | `CMD-SHELL immich-healthcheck` | `curl -fsS -m 2 http://$IMMICH_HOST:$IMMICH_PORT/api/server/ping` (defaults `localhost:2283`), fails unless body equals `{"res":"pong"}`. Skips cleanly (exit 0) if the `api` worker is excluded via `IMMICH_WORKERS_INCLUDE/EXCLUDE`. |
| `immich-machine-learning` | `CMD-SHELL python3 healthcheck.py` | `GET /ping` on `IMMICH_PORT` (default 3003), 200 required, 2s timeout. |
| `database` | `CMD-SHELL /usr/local/bin/healthcheck.sh` | Postgres readiness (incl. checksum verification); `Interval 300s`, `StartPeriod 300s`, `StartInterval 5s`. The `start_interval` field needs **Docker Engine >= 25**; docs note that on older engines you must comment it out (TrueNAS 25.10 ships a new enough engine; our other stacks already rely on modern compose behavior). |
| `redis` (Valkey) | inline in compose: `redis-cli ping` | Standard. |

Operational notes:

- `/api/server/ping` is a real unauthenticated endpoint returning `{"res":"pong"}` (enforced by the script), so it doubles as an external uptime probe through Caddy if ever needed.
- The default compose does **not** wire `depends_on: condition: service_healthy`; startup ordering is advisory. Immich tolerates the DB/Valkey coming up late (it retries), so this is by design.
- Removing the host port publish (section 2) does not affect any of these checks.

## Homelab fit summary (derived)

- Stack dir `truenas/apps/immich/`: start from the v3.1.0 release compose, strip `ports:`, add the external Caddy network + `caddy.reverse_proxy "{{upstreams 2283}}"` label on `immich-server`, keep `restart: always`, keep image healthchecks as shipped.
- Dockhand env: `DB_PASSWORD` (secret, plain env, expect the daemon-restart loss caveat), `IMMICH_VERSION=v3.1.0` (exact pin), `UPLOAD_LOCATION`, `DB_DATA_LOCATION`, `TZ`. No JWT secret, no ML URL, no Redis config needed.
- If we want real client IPs in Immich, add `IMMICH_TRUSTED_PROXIES` covering the shared Caddy network subnet; otherwise cosmetic.
- Upgrades are a two-line change plus Dockhand Deploy; expect startup migration pauses; `immich-admin` is the break-glass tool.

## Confidence notes

- **Doc-backed** (docs.immich.app + release notes, viewed 2026-08-19): 4-service compose shape and its ports; release-asset install flow; example.env contents; env-var list and semantics incl. `DB_URL` override, Redis vars, `IMMICH_PORT`/`IMMICH_HOST`, `IMMICH_TRUSTED_PROXIES`, `IMMICH_ALLOW_SETUP`, `_FILE` secret variants, recreate-to-apply; reverse-proxy requirements (four headers, big uploads, timeouts, websockets, no sub-path, `/.well-known/immich`); verbatim Caddy example; Traefik-label example; tag/metatag semantics and version policy; upgrade commands; startup-migration behavior and reindex waiting; VectorChord migration scope and the 1.133.0 floor; healthchecks-in-image statement; `immich-admin` command set; v3.0.0 breaking-change characterization; TrueNAS page is the ix catalog app (community-maintained), not compose.
- **Source/image-verified** (Immich repo @ `8aa95c6` and GHCR v3.1.0 image configs): server healthcheck script behavior and `{"res":"pong"}`; ML healthcheck behavior; Postgres healthcheck binary + intervals; server/ML/postgres ExposedPorts; `IMMICH_MACHINE_LEARNING_URL` default `http://immich-machine-learning:3003`; `IMMICH_MACHINE_LEARNING_ENABLED=false`; zero `JWT_SECRET` references; `trust proxy` = loopback + `IMMICH_TRUSTED_PROXIES` default `['linklocal','uniquelocal']`; immich-proxy/immich-web GHCR packages unpublished (all checked tags 404); historical topology (v1.92.0, v2.0.0, v2.4.0, v3.0.0 compose assets).
- **Community-sourced**: Caddy-on-shared-docker-network addressing (`immich-server:2283`) from discussion #7164 and r/immich `1cdeg47`; subdomain-only gotcha from issue #11666.
- **Undetermined**: exact release announcing immich-proxy's removal (v1.92.0 is inferred from that release's compose asset, not from a notes quote); an official release-cadence statement (characterization here is from observed release dates only); any official stance on auto-updaters (no current doc guidance); Caddy config-reload websocket disconnects are documented Caddy behavior but Immich docs do not discuss it.

## Sources

- Official compose asset (v3.1.0): [releases/latest/download/docker-compose.yml](https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml) and [example.env](https://github.com/immich-app/immich/releases/latest/download/example.env)
- Docs: [install/docker-compose](https://docs.immich.app/install/docker-compose) · [install/environment-variables](https://docs.immich.app/install/environment-variables) · [install/upgrading](https://docs.immich.app/install/upgrading) · [administration/reverse-proxy](https://docs.immich.app/administration/reverse-proxy) · [administration/server-commands](https://docs.immich.app/administration/server-commands) · [administration/maintenance-mode](https://docs.immich.app/administration/maintenance-mode) · [install/truenas](https://docs.immich.app/install/truenas)
- Source (v3.1.0 build, commit `8aa95c6`): [server/src/config.ts](https://github.com/immich-app/immich/blob/main/server/src/config.ts) · [server/src/app.common.ts](https://github.com/immich-app/immich/blob/main/server/src/app.common.ts) · [server/src/repositories/config.repository.ts](https://github.com/immich-app/immich/blob/main/server/src/repositories/config.repository.ts) · [server/bin/immich-healthcheck](https://github.com/immich-app/immich/blob/main/server/bin/immich-healthcheck) · [machine-learning/scripts/healthcheck.py](https://github.com/immich-app/immich/blob/main/machine-learning/scripts/healthcheck.py)
- Releases: [v3.1.0](https://github.com/immich-app/immich/releases/tag/v3.1.0) · [v3.0.0](https://github.com/immich-app/immich/releases/tag/v3.0.0) (notes) · [v3 migration blog](https://immich.app/blog/v3-migration) · [GitHub releases API](https://github.com/immich-app/immich/releases)
- Images inspected via GHCR registry API: `ghcr.io/immich-app/immich-server:v3.1.0`, `immich-machine-learning:v3.1.0`, `postgres:14-vectorchord0.4.3-pgvectors0.2.0`; 404 checks on `immich-proxy`/`immich-web`
- Caddy: [reverse_proxy directive docs](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- Community: [discussion #7164 (Caddy + dockerized Immich)](https://github.com/immich-app/immich/discussions/7164) · [issue #11666 (subdomain-only)](https://github.com/immich-app/immich/issues/11666) · [r/immich Caddy thread](https://www.reddit.com/r/immich/comments/1cdeg47/a_caddy_question_how_to_get_it_working_with_immich/)
