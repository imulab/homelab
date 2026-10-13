# Hermes gateway → Strata custom endpoint (Dockhand compose)

> Resolves wayfinder ticket #48. Compiled 2026-10-13 against the official
> Hermes docs (hermes-agent.nousresearch.com) and this repo's own files
> (`truenas/apps/README.md`, `truenas/apps/strata/{compose.yaml,.env.example,README.md}`,
> `research/strata-endpoint-api-surface.md`).

## TL;DR

- **Image** `nousresearch/hermes-agent`; pin by **digest** (`@sha256:…`) for an
  exact pin, or a versioned `X.Y.Z` tag for a trackable stable (avoid `latest` /
  `main`). **Command** is `gateway run`. In the official s6 image the gateway is
  supervised automatically — **keep supervision** (do not pass `--no-supervise`);
  it matters even for a single profile.
- **Custom endpoint** is a `model:` block: `provider: custom`, `base_url:
  http://strata:8080/v1`, `default: <id from Strata>`, `api_mode:
  chat_completions`. The model id goes in **`model.default`** (alias
  `model.model`) and must be read from Strata at deploy time — **not guessed**
  (see §2).
- **`STRATA_ALLOWED_HOSTS`** must be extended: the gateway sends `Host:
  strata:8080`, so add the container name `strata` (or `*`). No API key is
  needed either side (keyless Strata ⇒ `_authorized()` is always `True`).
- **SQLite WAL**: `database.journal_mode: delete` is **NOT** needed for a native
  ZFS bind mount on TrueNAS — that caveat is only for virtiofs / 9p / drvfs
  (VM-boundary) mounts.
- **Undetermined** (not resolvable from the listed sources): the exact current
  `X.Y.Z` tag / digest value; the exact Strata model id string; whether Strata's
  `host_allowed()` strips the `:8080` port before matching; whether the Hermes
  image ships a Dockerfile `HEALTHCHECK`.

---

## 1. Image, pin, command, supervision

All of this is **doc-backed** ([Hermes Docker Setup](https://hermes-agent.nousresearch.com/docs/user-guide/docker)).

- **Image**: `nousresearch/hermes-agent`. Channels: `latest` / `stable` (stable
  release gate), `main` (development), `X.Y.Z` (versioned stable). The docs say
  explicitly: *"Use a digest for an exact deployment pin."* So pin
  `nousresearch/hermes-agent@sha256:<digest>` for reproducibility, or a `X.Y.Z`
  tag if you want a stable line you can bump deliberately. The specific digest /
  current version is **undetermined** from the listed sources — fill it at
  deploy time (`docker pull` the tag, then record the digest).
- **Command**: `command: gateway run` is the correct persistent-gateway entry
  point.
- **s6 supervision**: inside the official image, `gateway run` is *"automatically
  supervised by s6-overlay"* — a crash is restarted within seconds, the CMD
  process itself is a `sleep infinity` heartbeat, and `docker stop` still shuts
  down cleanly. For a **single-profile** deploy this still matters: the default
  profile's gateway is always registered as a supervised service on first boot,
  and the dashboard (when enabled) is supervised alongside it. The docs call the
  supervised default *"strictly better"* for production; the opt-out
  (`--no-supervise` / `HERMES_GATEWAY_NO_SUPERVISE=1`) is only for CI smoke
  tests that want the container exit code. **Do not opt out here.**

## 2. Custom-endpoint config (Hermes → Strata)

The `model:` mapping has the sub-keys **`provider`**, **`default`** (alias
`model` — both `model: {default: x}` and `model: {model: x}` are identical),
**`base_url`**, and **`api_mode`** (**doc-backed**, [Configuring Models](https://hermes-agent.nousresearch.com/docs/user-guide/configuring-models)
and [AI Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)).

The bare **Custom Endpoint** form (the vision example from the providers page,
adapted):

```yaml
# /opt/data/config.yaml  (mounted from the TrueNAS dataset)
model:
  provider: custom
  base_url: http://strata:8080/v1
  default: <STRATA_MODEL_ID>        # read from Strata, do not guess (below)
  api_mode: chat_completions        # OpenAI-compatible /v1/chat/completions
  context_length: 262144            # pin to Strata's deployed window
  supports_vision: true             # Strata serves VISION=cpu
```

Point-by-point:

- **`base_url`**: `http://strata:8080/v1`. `strata` is the `container_name`
  (and service name) of the Strata stack on the external `proxy` network; `8080`
  is Strata's *container* port (the host port `27182` is only the LAN fallback).
  Both stacks join `proxy`, so Docker's embedded DNS resolves `strata` to the
  Strata container. **doc-backed** for the URL shape ([providers](https://hermes-agent.nousresearch.com/docs/integrations/providers));
  **repo-backed** for `strata`/`8080` (`truenas/apps/strata/compose.yaml`,
  `container_name: strata`, `ports: …:8080`, `networks: [proxy]`).
- **`model.default`** (the id): this is the **config key** to set; the value is
  **not provided here by design**. Strata reports what it is running via
  `GET /v1/status` (and lists models via `GET /v1/models`) — read the id from
  there at deploy time and put that exact string here. Guessing an id is the one
  thing the ticket forbids. **undetermined** (value lives only in the running
  Strata engine).
- **provider / transport**: `provider: custom` + `api_mode: chat_completions`.
  Strata natively implements OpenAI-style `/v1/chat/completions` with real
  `tool_calls` (non-streamed and streamed) — see
  [research/strata-endpoint-api-surface.md](/Users/imulab/workspace/homelab/research/strata-endpoint-api-surface.md)
  — so the OpenAI-compatible `chat_completions` transport carries Hermes' tools
  with **no Hermes-side tool-calling workaround** needed. **doc-backed**
  (providers page lists `transport: chat_completions` / `anthropic_messages` /
  `codex_responses`; `chat_completions` is the default for a `/v1` base URL).
- **≥64k context**: the providers page states *"Hermes Agent requires at least
  64,000 tokens of context for agent use with tools. Smaller windows are rejected
  at startup"*, and that this startup refusal applies to local endpoints
  including *"Docker service names"*. Strata serves `262144`, so the requirement
  is satisfied — but Hermes auto-detects the window from `/v1/models` and a miss
  would reject the endpoint. Pin `context_length: 262144` to match the deployed
  `STRATA_CONTEXT`. **doc-backed** (≥64k requirement, `model.context_length` is
  the highest-priority explicit pin) + **repo-backed** (`STRATA_CONTEXT=262144`).
- **`supports_vision`**: set `true` because Strata runs `VISION=cpu`; without it,
  Hermes pre-describes images through `vision_analyze` instead of sending native
  `image_url` parts. **doc-backed** (`model.supports_vision: true` key) +
  **repo-backed** (`STRATA_VISION=cpu`).
- **No key**: a keyless custom endpoint omits `api_key` — the docs note Hermes
  then uses *"no-key-required"* for keyless local servers. **doc-backed**.

## 3. Compose shape (repo conventions)

Repo conventions are **repo-backed** (`truenas/apps/README.md`, "Per-stack
conventions", decision #5): one `compose.yaml`, one name everywhere (dir =
stack = container = service name on `proxy`), `networks: [proxy]` `external:
true`, `restart: unless-stopped`, a `.env.example`, and *"add a `healthcheck:`
when the image ships one; never fabricate one."*

- **`/opt/data`** is the Hermes state root (config, `.env`, sessions, memories,
  skills, `state.db`) — bind-mount it from a TrueNAS dataset (the `strata` stack
  uses `zfs create -p puddle/apps/strata`, so mirror that with
  `puddle/apps/hermes`). **doc-backed** (Docker page: `/opt/data` is *"the single
  source of truth for all Hermes state"*).
- **Port 8642** = gateway OpenAI-compatible API server + health endpoint
  (**doc-backed**). To serve it beyond container loopback the docs require
  `API_SERVER_ENABLED=true`, `API_SERVER_HOST=0.0.0.0`, and an `API_SERVER_KEY`
  (min 8 chars).
- **Port 9119** = dashboard, **only** when `HERMES_DASHBOARD=1` (**doc-backed**).
  Omit unless the dashboard is wanted.
- **Healthcheck**: the docs describe an HTTP *health endpoint* on 8642 but do
  **not** document a Dockerfile `HEALTHCHECK` in the image. Per repo convention,
  do not fabricate one. **undetermined** whether the image ships one; the safe
  repo-consistent move is to leave `healthcheck:` out.

## 4. Gateway → Strata auth and Host header

**repo-backed** (`truenas/apps/strata/{compose.yaml,.env.example,README.md}`).

- **Keyless Strata needs no key from Hermes.** Strata's `_authorized()` returns
  `True` for everything when no `API_KEY` is set, and Hermes' keyless custom
  endpoint sends no `Authorization` header. Nothing to configure on either side
  for auth.
- **Host header**: a request to `http://strata:8080/v1` sends `Host:
  strata:8080` (host **plus port**, because `8080` is not the default). This is
  an HTTP fact, not a guess.
- **`STRATA_ALLOWED_HOSTS` must change.** Today it is
  `STRATA_ALLOWED_HOSTS=strata.absurdlab.dev`, which will **not** match
  `strata:8080`. The Host check runs *only when no API key is set* (which is the
  deployed keyless case), so the gateway's requests would be rejected with the
  DNS-rebinding `403`. Add the container name:
  `STRATA_ALLOWED_HOSTS=strata.absurdlab.dev,strata`. This is a change to the
  **Strata** stack's env (Dockhand panel), not the Hermes stack.
- **Port-stripping nuance (undetermined)**: the repo files document that the
  check accepts a bare name, a leading-dot suffix, or `*`, but they do not say
  whether `host_allowed()` strips the `:8080` before comparing. If the engine
  still logs `Host 'strata:8080' is not allowed` after adding `strata`, add
  `strata:8080` too, or use `*`. Since the Host check is explicitly *not access
  control* (it only inspects a header; network placement is the real control),
  `*` is a deterministic, acceptable fallback (the server just warns at start).

## 5. SQLite WAL on a ZFS bind mount

**doc-backed** ([Hermes Docker Setup](https://hermes-agent.nousresearch.com/docs/user-guide/docker),
"Filesystem requirements for `state.db`").

`database.journal_mode: delete` is **NOT needed**. The WAL caveat exists only for
bind mounts that cross a VM boundary — **virtiofs** (Docker Desktop / Podman on
macOS, OrbStack) and **9p / drvfs** (Docker Desktop on Windows) — because WAL's
`state.db-shm` shared memory must be coherent between processes, and those
transports break it. A TrueNAS ZFS dataset bind-mounted into a container is a
**native kernel filesystem** (like the docs' own "native volume … supports WAL
normally"), and the docs state the detection *"does not classify NFS, SMB, or
generic FUSE mounts"* — ZFS native is none of those either. Leave WAL (the
default) alone.

## Sources

- [Hermes Docker Setup](https://hermes-agent.nousresearch.com/docs/user-guide/docker) —
  image channels/digest pinning, `gateway run`, s6 supervision, `/opt/data`,
  ports 8642/9119, `API_SERVER_*`, `HERMES_DASHBOARD`, and the SQLite WAL /
  virtiofs / 9p caveat.
- [LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) —
  "Custom Endpoint" + "Named Custom Providers" (`api`/`base_url`, `transport`,
  `key_env`/`api_key`/`key_cmd`), `model.supports_vision`, `model.context_length`
  and the ≥64k-context requirement, and the context auto-detection chain.
  (Raw source of the same page:
  [NousResearch/hermes-agent `website/docs/integrations/providers.md`](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/integrations/providers.md),
  fetched because the rendered page truncates before those sections.)
- [Configuring Models](https://hermes-agent.nousresearch.com/docs/user-guide/configuring-models) —
  `model:` sub-keys (`provider`, `default`, `base_url`, `api_mode`) and the
  `default`/`model` key alias.
- Local: [`truenas/apps/README.md`](/Users/imulab/workspace/homelab/truenas/apps/README.md)
  (stack conventions), [`truenas/apps/strata/compose.yaml`](/Users/imulab/workspace/homelab/truenas/apps/strata/compose.yaml),
  [`truenas/apps/strata/.env.example`](/Users/imulab/workspace/homelab/truenas/apps/strata/.env.example),
  [`truenas/apps/strata/README.md`](/Users/imulab/workspace/homelab/truenas/apps/strata/README.md)
  (Strata endpoint: container 8080 / host 27182, `STRATA_ALLOWED_HOSTS`,
  keyless-by-default, `API_KEY` optional, Host-check semantics),
  [`research/strata-endpoint-api-surface.md`](/Users/imulab/workspace/homelab/research/strata-endpoint-api-surface.md)
  (native OpenAI-style tool calling, `/v1/status`, `/v1/models`).

---

## Prototype #50 starter (copy these)

### `truenas/apps/hermes/compose.yaml`

```yaml
services:
  hermes:
    # Pin a digest at deploy time (docker pull the tag, record the digest).
    # A versioned X.Y.Z tag is the trackable alternative; avoid latest/main.
    image: nousresearch/hermes-agent@sha256:<digest>
    container_name: hermes
    restart: unless-stopped
    command: gateway run          # s6-supervised inside the image — leave as is
    environment:
      # OpenAI-compatible API server on 8642 (docs require all three to serve
      # beyond container loopback). Optional if you only use chat platforms.
      API_SERVER_ENABLED: "true"
      API_SERVER_HOST: 0.0.0.0
      API_SERVER_KEY: ${HERMES_API_SERVER_KEY:?set in Dockhand - openssl rand -hex 32}
      # Dashboard: enable only if wanted, and only then publish 9119.
      # HERMES_DASHBOARD: "1"
    ports:
      - "8642:8642"               # gateway API + health endpoint
      # - "9119:9119"             # only when HERMES_DASHBOARD=1
    volumes:
      # Create first: zfs create -p puddle/apps/hermes ; then place config.yaml there.
      - ${HERMES_DATA_PATH:?set in Dockhand}:/opt/data
    networks:
      - proxy

networks:
  proxy:
    external: true
```

### `truenas/apps/hermes/.env.example` (documentation only)

```ini
# Dockhand stack `hermes`, compose path truenas/apps/hermes/compose.yaml.
# Values live in Dockhand's env-var panel, never a real .env (gitignored).

# /opt/data root (config.yaml, .env, sessions, memories, skills, state.db).
# Create pre-deploy:  zfs create -p puddle/apps/hermes
HERMES_DATA_PATH=/mnt/puddle/apps/hermes

# API server key (min 8 chars): openssl rand -hex 32
# Plain env var (baked at create time) so it survives a daemon-initiated
# restart — mirroring the strata stack's "no key-icon secret on a
# restart: unless-stopped stack" reasoning. An unset key fails closed.
HERMES_API_SERVER_KEY=
```

### `/opt/data/config.yaml` (place in the dataset; model id is filled at deploy)

```yaml
model:
  provider: custom
  base_url: http://strata:8080/v1
  default: <STRATA_MODEL_ID>    # from GET http://strata:8080/v1/status  (do not guess)
  api_mode: chat_completions
  context_length: 262144        # Strata's deployed window; satisfies Hermes' >=64k gate
  supports_vision: true         # Strata VISION=cpu
```

### Strata stack change (required, separate stack)

```ini
# In the strata Dockhand env panel (not the hermes stack):
STRATA_ALLOWED_HOSTS=strata.absurdlab.dev,strata
# If the engine still rejects "Host 'strata:8080'", use:  strata,strata:8080
# or "*" (acceptable — the Host check is not access control).
```
