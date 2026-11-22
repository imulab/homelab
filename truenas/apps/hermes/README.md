# Hermes deployment runbook

A Dockhand-managed Hermes gateway (`nousresearch/hermes-agent`, pinned) on the
Strata endpoint, serving life admin and occasional coding execution for the
operator and family. The primary interface is email threads at a dedicated
`hermes@absurdlab.dev` on iCloud+. The web dashboard and keyed API server are
exposed on the LAN. Voice and smart home are a later slice.

Model, channel, and other agent settings are configured in the Hermes app after
the first Deploy. The dashboard and the container's `hermes` CLI write them to
the mounted `/opt/data` volume (`config.yaml` and `.env`). That volume is the
configuration store. This repo carries only the deployment wiring.

| | |
|---|---|
| Image | `nousresearch/hermes-agent:rc.15-v0.21.7-amd64@sha256:3140cc382d3f…` (v0.21.7) |
| Model | Strata `swift-1.5-iq3_s` (Swift 1.5 IQ3_S), 262144 context, vision, API key |
| Channel | Email (IMAP/SMTP, iCloud app-specific password) |
| Admin | operator only (family added later) |
| Ports | 9119 dashboard (Caddy `hermes.absurdlab.dev`), 8642 API (LAN, keyed) |
| Config | `/opt/data/config.yaml` and `/opt/data/.env`, written by the app |

## 1. Prerequisites

- Strata 0.2.17 is deployed and its `STRATA_API_KEY` is set. Every `/v1`
  request needs that key. Hermes reaches Strata container-to-container at
  `http://strata:8080/v1` over the external `proxy` network.
- Strata serves the model as `swift-1.5-iq3_s` at 262144 context with vision.
  Confirm on the host:

  ```sh
  curl -s -H "Authorization: Bearer $STRATA_API_KEY" \
    http://192.168.20.11:27182/v1/models
  ```

- The iCloud mailbox (section 2) and the dataset (section 3) exist before the
  first Deploy.

## 2. iCloud+ custom domain and app-specific password

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
5. Record these for section 5: address, app-specific password, and the iCloud
   mail endpoints `imap.mail.me.com:993` / `smtp.mail.me.com:587`.

## 3. Dataset

```sh
zfs create -p puddle/apps/hermes/data
```

The supervised gateway runs as UID 10000 (`hermes`), so own the dataset to
match:

```sh
chown -R 10000:10000 /mnt/puddle/apps/hermes/data
```

`docker exec ... hermes` goes through the image's privilege-drop shim, which
runs the CLI as the same user. CLI writes stay owned correctly. `puddle/apps/*`
is already in the house snapshot tree.

## 4. Deploy in Dockhand

1. Add the hostname per `apps/README.md`: in TrueNAS **Network → DHCP and DNS →
   Hostnames** add `hermes.absurdlab.dev` → the Caddy box LAN IP, then refresh
   Passwall's DNS with the SIGHUP snippet.
2. Deploy stack **`hermes`** with compose path
   `truenas/apps/hermes/compose.yaml` and the values from `.env.example`. The
   three dashboard and API secrets are required deploy inputs; set them in the
   Dockhand env-var panel with the key-icon toggle. Hermes does not start the
   dashboard without them.
3. The first start seeds `/opt/data`. Open `https://hermes.absurdlab.dev`
   and sign in as the dashboard user.

## 5. Configure in the app

### Model

Register the Strata endpoint in the dashboard. No container command is needed.

1. Add the API key. Open **Keys → Add custom key** in the dashboard:

   | Field | Value |
   |---|---|
   | Name | `HERMES_CUSTOM_STRATA_8080_API_KEY` |
   | Value | the same `STRATA_API_KEY` as the Strata stack |

   The key goes to `/opt/data/.env`. That file is mode 600; the key does not go
   in `config.yaml`.

2. Add the model. Open **Config → YAML** and replace the existing `model:` block
   (it currently points at OpenRouter) with:

   ```yaml
   model:
     default: "swift-1.5-iq3_s"
     provider: "custom"
     base_url: "http://strata:8080/v1"
     api_key: "${HERMES_CUSTOM_STRATA_8080_API_KEY}"
     api_mode: "chat_completions"
     context_length: 262144
     supports_vision: true
   ```

   Save the page. The YAML tab writes the whole `config.yaml`, so leave the rest
   of the file as it is.

3. New sessions use the model. The Models page lists the endpoint as
   "Custom endpoint" and can switch back to it later. A changed endpoint, base
   URL, or unlisted model id needs another edit here.

The CLI equivalent is `docker exec -it hermes hermes model` if the dashboard is
unreachable.

### Email

Open the dashboard at `https://hermes.absurdlab.dev`, then **Channels → Email →
Configure**. Fill the four required fields:

| Field | Value |
|---|---|
| Email address | `hermes@absurdlab.dev` |
| Email password | iCloud app-specific password |
| IMAP host | `imap.mail.me.com` |
| SMTP host | `smtp.mail.me.com` |

The form also carries `EMAIL_ALLOWED_USERS` (the reach gate) and
`EMAIL_AUTHSERV_ID` (sender authentication). Fill the allowlist with the
operator address, then click **Save & enable**, run **Test**, and use
**Restart gateway** on the page. The page writes the credentials to
`/opt/data/.env` and `platforms.email.enabled` to `/opt/data/config.yaml`.

Sender authentication is on by default. If `EMAIL_AUTHSERV_ID` is blank, mail
from allowlisted senders is dropped. Send a test message and check the log:

```sh
docker logs hermes 2>&1 | grep -i "unauthenticated From"
```

If it drops, read the topmost `Authentication-Results` header of a delivered
message and put its authserv-id in that field, then restart the gateway. Adding
`EMAIL_TRUST_FROM_HEADER=true` in **Keys** accepts the spoofing risk instead;
prefer the pin.

## 6. Verify

```sh
# Endpoint and model from the app's own config
docker exec hermes hermes config get model --json

# The Strata key is in .env, not config.yaml
docker exec hermes grep -c '^HERMES_CUSTOM_STRATA_8080_API_KEY=' /opt/data/.env   # expect 1

# Email config is app data, not compose env
# (remove any old EMAIL_* rows from the Dockhand panel first)
docker exec hermes printenv EMAIL_ADDRESS   # expect no output
docker exec hermes grep -c '^EMAIL_' /opt/data/.env   # expect 4 or more

# Gateway came up healthy
docker logs hermes 2>&1 | tail -40

# Dashboard reachable and auth gate on (expect a login)
#   https://hermes.absurdlab.dev
# The Models page shows "custom · swift-1.5-iq3_s".

# API server keyed (expect 401 without the key)
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.20.11:8642/v1/models

# Strata serves the model (run on the host)
curl -s -H "Authorization: Bearer $STRATA_API_KEY" \
  http://192.168.20.11:27182/v1/models
```

End-to-end acceptance: the operator's email gets a threaded reply; a family
address not in `EMAIL_ALLOWED_USERS` is ignored; and a task that needs a tool
produces a native Strata `tool_calls` round-trip, not parsed text. After a host
reboot the model and email settings are still present because they live in
`/opt/data`.

## 7. Day 2

- **Add a family user.** Append their address to `EMAIL_ALLOWED_USERS` on the
  Channels page, then add the admin tier in **Config → YAML** under
  `platforms.email`:

  ```yaml
  platforms:
    email:
      enabled: true
      extra:
        allow_admin_from:
          - operator@absurdlab.dev
        user_allowed_commands:
          - status
          - model
          - usage
  ```

  Save, then confirm the tier with `/whoami` from each address.

- **Change the model.** Use the dashboard Models page for a model it lists. A
  new endpoint, base URL, or model id that the picker does not list re-runs
  `hermes model`.
- **Upgrade.** Change the image tag and digest in `compose.yaml`, then Deploy.
  `/opt/data` persists across pulls.
- **Secrets after reboot.** Dashboard auth and API server keys remain Dockhand
  inputs. A daemon-initiated restart can drop them (decision #8), so redeploy
  the stack after a host reboot. Model and email settings are unaffected
  because they live in `/opt/data`.
- **Voice and smart home.** Deferred. Hermes Voice Mode and the Home Assistant
  plugin are the assumed path; no config in this slice.
