# Hermes — deployment runbook

A Dockhand-managed **Hermes** gateway (`nousresearch/hermes-agent`, pinned) on the
Strata Qwen3.8 endpoint, serving life-admin + occasional coding execution for the
operator and family. Primary interface: **email threads** at a dedicated
`hermes@absurdlab.dev` on iCloud+. Web dashboard + keyed API server are exposed on
the LAN; voice and smart-home are a later slice.

| | |
|---|---|
| Image | `nousresearch/hermes-agent:rc.15-v0.21.7-amd64@sha256:3140cc382d3f…` (v0.21.7) |
| Model | Strata custom endpoint `http://strata:8080/v1` (256K, native tool calling) |
| Channel | Email (IMAP/SMTP, iCloud app-specific password) |
| Admin | operator only (family added later) |
| Ports | 9119 dashboard (Caddy `hermes.absurdlab.dev`), 8642 API (LAN, keyed) |

## 1. Prerequisites

- Strata deployed and `STRATA_ALLOWED_HOSTS` includes the container name `strata`
  (`strata.absurdlab.dev,strata` in truenas/apps/strata/.env.example). Hermes
  reaches it container-to-container; no Caddy hop. Confirm the live Dockhand
  value matches before the first Deploy.
- The iCloud mailbox (section 2) and the dataset (section 3) exist before the
  first Deploy.

## 2. iCloud+ custom domain + app-specific password

This is the one manual, human-only step. It makes iCloud the mail provider for
`absurdlab.dev` (adds MX/DKIM records), so be aware it touches the domain's DNS.

1. Confirm **iCloud+** (paid) is active on your Apple ID.
2. Add the custom domain: on iPhone/Mac **Settings → [your name] → iCloud →
   Custom Email Domain** (or [icloud.com/settings](https://www.icloud.com/settings)).
   Add `absurdlab.dev`; Apple shows the **DNS records** to publish. Add them at
   your DNS provider/registrar (you own `absurdlab.dev`).
3. Create the address **`hermes@absurdlab.dev`** under that domain.
4. Generate an **app-specific password** for Hermes:
   [account.apple.com](https://account.apple.com) → **Sign-In and Security →
   App-Specific Passwords** → name it `Hermes`, copy the 4-group password.
5. Record these for section 4: address, app-specific password, and the iCloud
   mail endpoints `imap.mail.me.com:993` / `smtp.mail.me.com:587`.

## 3. Datasets

```sh
zfs create -p puddle/apps/hermes/data
```

The official image runs as the non-root `hermes` user. On first start, note the
runtime UID/GID it expects and chown `puddle/apps/hermes/data` to match (see the
image's own docs); the `strata`/`comfyui` stacks run as root and are not a
precedent here. `puddle/apps/*` is already in the house snapshot tree.

## 4. Seed /opt/data (before first Deploy)

Place two files in `puddle/apps/hermes/data/`:

- `config.yaml` — copy `truenas/apps/hermes/config.yaml.example`, fill
  `<MODEL_ID>` (from `curl http://192.168.20.11:27182/v1/status` on the host) and
  your email in `allow_admin_from`.
- `.env` — the secrets from `.env.example`'s "/opt/data/.env secrets" block:
  `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_IMAP_HOST`, `EMAIL_SMTP_HOST`,
  `EMAIL_IMAP_PORT=993`, `EMAIL_SMTP_PORT=587`, `EMAIL_POLL_INTERVAL=15`,
  `EMAIL_ALLOWED_USERS`, `API_SERVER_KEY`, `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD`,
  `HERMES_DASHBOARD_BASIC_AUTH_SECRET`.
- `SOUL.md` — optional persona file in `/opt/data`; name the agent "Hermes" here
  (spec #8). Hermes reads it as the agent's identity/personality.

**Verify at bring-up** whether Hermes reads the three gateway/dashboard secrets
from `/opt/data/.env` or requires them as process env; if the latter, set them as
**plain** env in Dockhand (never key-icon) and expect a manual redeploy after a
host reboot (decision #8).

## 5. Deploy in Dockhand

1. Add the hostname (per `apps/README.md`): in TrueNAS **Network → DHCP and DNS →
   Hostnames** add `hermes.absurdlab.dev` → the Caddy box LAN IP, then refresh
   Passwall's DNS (the SIGHUP snippet in `apps/README.md`).
2. Deploy stack **`hermes`** (compose path `truenas/apps/hermes/compose.yaml`),
   with the non-secret values from `.env.example` in the env-var panel (key-icon
   toggles **OFF**).

## 6. Verify

```sh
# Host ports free of the TrueNAS UI / other stacks
ss -lntp | grep -E ':(9119|8642)\b'

# Gateway came up healthy (no fabricated healthcheck; watch the log)
docker logs hermes 2>&1 | tail -40

# Dashboard reachable + auth gate on (expect a login, not an open page)
#   https://hermes.absurdlab.dev

# API server keyed (expect 401 without the key)
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.20.11:8642/v1/models

# Email round-trip: send hermes@absurdlab.dev a message from your address,
# expect a threaded reply. Confirm the tier took effect:
#   /whoami   (from your email) -> admin
```

End-to-end acceptance: a family member's email is refused unless listed in
`EMAIL_ALLOWED_USERS`; your email gets a threaded reply; and a task that needs a
tool produces a native Strata `tool_calls` round-trip (not parsed text).

## 7. Day-2

- **Add a family user:** append their address to `EMAIL_ALLOWED_USERS` in
  `/opt/data/.env`, and add them as a regular user (not `allow_admin_from`) in
  `config.yaml`; restart the gateway (`docker restart hermes`).
- **Upgrade:** change the image tag+digest in `compose.yaml`, Deploy. State lives
  in `/opt/data`, so it persists across pulls.
- **Secrets after reboot:** if any secret must be a process env var, redeploy the
  stack from Dockhand after a host reboot (decision #8).
- **Voice / smart-home:** deferred. Hermes's Voice Mode and Home Assistant plugin
  are the assumed path; no config in this slice.
