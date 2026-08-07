# Dockhand capabilities briefing

> Resolves wayfinder ticket #2. Compiled 2026-08-07 via `/research` probe.
> Canonical identity: **Dockhand by Finsys**, [dockhand.pro](https://dockhand.pro) · [docs.dockhand.io](https://docs.dockhand.io) · [github.com/Finsys/dockhand](https://github.com/Finsys/dockhand). (Note: the "HeavyOps / dockhand.io by" framing is incorrect — no such vendor/domain appears in the public record.)

## What it is

A Docker management UI / orchestrator that drives **Docker Compose**. Not Swarm, not Kubernetes, not a novel declarative format. Ships as a TrueNAS 25.10 community app; requires a host mount of `/var/run/docker.sock`, runs as UID/GID 568, default UI port 30328. SvelteKit/Bun + SQLite (or Postgres), Wolfi base image. Mental model: closer to Portainer/Dockge than to Argo/Flux.

## Repo layout

Dockhand consumes **plain `docker-compose.yml` files inside a git repo** — no chart format, no custom manifest.

- **One git repo backs many independent stacks.** Each Dockhand "Git Stack" is configured with: repository + branch, a **compose file path** (default `docker-compose.yml`), and an optional **context directory**.
- "Multiple stacks can use the same Git repository with different compose paths."
- The whole directory containing the compose file is cloned — co-located config files (Caddyfile, `.env`, scripts) travel with it.
- **Diff is scoped per stack:** each stack only redeploys when files inside *its* compose directory change.

Mono-repo layout this implies for us:

```
truenas/apps/
├── caddy/
│   ├── docker-compose.yml
│   └── Caddyfile
├── dashy/
│   └── docker-compose.yml
└── README.md          # ignored — outside any stack's context dir
```

## Reconciliation model — the key section

Three deploy triggers:
1. **Scheduled auto-sync** (cron expression per stack — **opt-in**)
2. **Webhook** from GitHub/GitLab/Gitea/Forgejo
3. **Manual "Deploy" button** in the UI (always forces redeploy)

**Change detection is `git diff`-based, not blind apply.** On any trigger, Dockhand diffs the files in the stack's compose directory; nothing changed → no deploy (unless "Force redeployment" is on).

**Manual workflow we want (confirmed supported):** leave the cron empty, don't wire a webhook → commits land but apply only when you click Deploy. This is exactly the "push config, manually ask Dockhand to apply" model.

- UI Deploy button: yes.
- CLI: **no**. UI + REST API only.
- Webhook to hit manually: yes, generated per-stack in the UI; default is diff-gated (skip if no change), force with "Force redeployment".
- REST endpoint exists for triggering stack sync; exact path not printed in the public manual (generated in UI).

**Caveat:** a commit that doesn't touch a stack's directory won't redeploy it even on a manual trigger unless Force is on.

## Secrets

Dockhand has its own secrets layer, **not** Docker Compose `secrets:`, sops, Vault, or TrueNAS credential integration.

- Mark any env var as a secret in the stack editor (key-icon toggle).
- **AES-256-GCM encrypted, stored only in the Dockhand DB** — never written to `.env` on disk.
- At deploy time, secrets are injected as **shell environment variables passed to the `docker compose` process**. Map them in compose: `environment: - CF_API_TOKEN=${CF_API_TOKEN}`.
- Encryption key lives at `$DATA_DIR/.encryption_key` (overridable via `ENCRYPTION_KEY`).

**Operational gotcha (doc-stated, repeatedly flagged):** "Docker-restarted containers will not have their secret values." On a daemon-initiated restart (host reboot, `restart: unless-stopped`, crash) the runtime-injected secret is gone → app comes up missing credentials. Fix: redeploy the stack from Dockhand. Real day-2 issue.

**Cloudflare API token pattern:**
1. Dockhand stack UI → add `CF_API_TOKEN`, paste token, toggle key icon.
2. Compose → `environment: - CF_API_TOKEN=${CF_API_TOKEN}`.
3. Back up `ENCRYPTION_KEY` and the DB — filesystem-only backups of compose files are insufficient.

## Networking

Standard Docker Compose networking — no special model.

- Per-stack networks follow whatever compose declares. Bridge (default), host, none, or user-defined custom. Standard `host_port:container_port/protocol` publishing.
- **Shared network for edge-proxy-to-backends: yes.** Create a docker network once (e.g. `proxy`), declare it `external: true` in both the proxy stack's compose and each backend's compose → proxy reaches backends by service name.
- Dockhand reads Traefik/Pangolon labels to surface a clickable public-URL pill; no equivalent Caddy label parsing documented (but none needed for our approach).
- Network graph view visualizes cross-stack service connections.

## Operational surface

- Config under `$DATA_DIR` (default `/app/data` in-container): `db/dockhand.db`, `.encryption_key`, `git-repos/`, `stacks/{env}/{stack}/`.
- Day-2: start/stop/restart containers and stacks, real-time logs, interactive `exec`, in-place file browser, image scanning (Grype + Trivy), prune/update cron Schedules, Activity log, Backups (restic-style; local/S3/B2/Azure/REST) that include stack files *and* secrets.
- Update a single stack: open it → Deploy (force); or push to git + cron/webhook; or hit the per-stack webhook.
- Per-stack deploy options: build images, disable build cache, re-pull images, force redeployment.
- Multi-host via "Environments" (local socket, remote TCP, or proprietary Hawser agent — outbound-only, NAT-friendly).
- Auth: local users, OIDC/SSO, MFA. RBAC/LDAP/audit-logging are Enterprise-only.

## Maturity / caveats

- **Version:** v1.0.40 (2026-07-31); repo created 2025-12-28, ~5.5k stars, actively pushed. Community reports it became "a lot less buggy and quite useable" around v1.0.13.
- **License:** Business Source License 1.1. Free for personal/internal-business/non-profit/education; converts to Apache 2.0 on **2029-01-01**. Pricing: Free (homelabs, unlimited envs), SMB $499/host/yr, Enterprise $1,499/host/yr (RBAC/LDAP/audit are Enterprise-gated).
- **TrueNAS app is community-train** — ix-systems does not validate or maintain it; support is forum-only.
- Known rough edges (community):
  - Secrets vanish on daemon-initiated restarts (see above).
  - Dockhand cannot self-update (unlike Dockge); update the Dockhand container out-of-band.
  - One report of an update "trapping" containers, needing CLI intervention.
- **vs real GitOps:** reconciliation is trigger-based + diff-gated, **not** a continuous control loop. No drift detection / auto-pruning of manual out-of-band changes by default.

## Confidence notes

- **Doc-backed:** compose-driven layout, multi-stack-per-repo, per-directory diff scoping, the three deploy triggers, opt-in auto-sync, manual Deploy button, force-redeploy, secrets model, restart caveat, env var list, data-dir layout, networking model, external networks, license, pricing, current version, no CLI, multi-host via Hawser.
- **Community-sourced:** v1.0.13 stability inflection, self-update limitation vs Dockge, trapped-container update report, Portainer/Dockge comparisons.
- **Undetermined from public docs:** exact per-stack webhook URL path; exact REST endpoint for triggering a stack sync by id; whether Compose `secrets:` or any external secret backend is supported (docs mention neither — treat as unsupported); whether any drift detection exists beyond trigger-based diff-gated deploys (none evident).

## Sources

[dockhand.pro](https://dockhand.pro) · [dockhand.pro/manual](https://dockhand.pro/manual/) · [docs.dockhand.io](https://docs.dockhand.io) · [github.com/Finsys/dockhand](https://github.com/Finsys/dockhand) · [apps.truenas.com/catalog/dockhand_community](https://apps.truenas.com/catalog/dockhand_community/) · [forums.truenas.com](https://forums.truenas.com/t/latest-dockhand-from-yaml-anyone/62957) · [r/truenas](https://www.reddit.com/r/truenas/comments/1svfdl6/anyone_else/) · [Terraform provider](https://registry.terraform.io/providers/kalebharrison/dockhand/latest/docs)
