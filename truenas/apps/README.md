# truenas/apps

Dockhand-managed stacks for the TrueNAS homelab. Each subdirectory is one
stack, applied via Dockhand's manual Deploy button (git is the source of
truth; nothing auto-deploys — decision #2).

## Layout

- `networks/proxy/` — bootstrap stack that creates the shared `proxy` docker
  network. Deploy once on a fresh box. Every service stack references `proxy`
  as `external: true`.
- `caddy/` — the edge proxy. Terminates Let's Encrypt TLS for `*.absurdlab.dev`
  (+ apex) via Cloudflare DNS-01, and reverse-proxies to backends by reading
  their Docker labels (caddy-docker-proxy).
- `<other>/` — backend stacks, one directory each.

## Per-stack conventions (decision #5)

- One `compose.yaml` per stack directory (`docker-compose.yml` is the Dockhand
  default, but this tree uses the Compose Spec name; set the compose path in
  Dockhand per stack).
- One name everywhere: directory name = Dockhand stack name = container name =
  service name on the `proxy` network. Caddy reaches backends by this name.
- Every stack joins `networks: [proxy]` as `external: true`.
- Every stack sets `restart: unless-stopped`. Note from decision #8:
  secret-bearing stacks come back up **missing their secrets** after a
  daemon-initiated restart (host reboot, dockerd crash) and must be
  redeployed from Dockhand to re-inject. The risk concentrates on `caddy`
  (holds `CF_API_TOKEN`); backends without secrets self-recover cleanly.
- Every stack carries a `.env.example` (documentation only — values live in
  Dockhand's encrypted DB, set via the Stack editor env-var panel with the
  key-icon toggle). Real `.env` files are gitignored at the repo root.
- Add a `healthcheck:` when the image ships one; never fabricate one.

## Add a new stack `<name>`

1. Create `truenas/apps/<name>/` with `compose.yaml` and `.env.example`.
2. In `compose.yaml`: join `networks: [proxy]` as `external: true`; set
   `container_name: <name>`; set `restart: unless-stopped`.
3. Document any secrets in `.env.example`; set their values in Dockhand's
   Stack editor env-var panel (key-icon toggle), never in git.
4. Declare the Caddy route via labels on the service (no edits to `caddy/`):
   ```yaml
   labels:
     caddy: <name>.absurdlab.dev
     caddy.reverse_proxy: "{{upstreams}}"
   ```
5. Add a photonicat dnsmasq entry: `<name>.absurdlab.dev` → Caddy box LAN IP
   (manual; decision #4), then refresh Passwall's DNS process as described
   below. Note: Clash Verge TUN mode on a client must be off for homelab
   domains to resolve, pending a bypass rule.
6. Push, then Deploy the stack in Dockhand.

## Refresh Photonicat DNS after adding a hostname

Photonicat runs two dnsmasq processes while Passwall DNS redirection is
enabled: OpenWrt's main dnsmasq and Passwall's private `dnsmasq_default`.
Passwall redirects port 53 to its private process. LuCI's **Save & Apply**
regenerates `/tmp/hosts` and reloads the main process, but the private process
can retain its previous hosts snapshot. Restarting only the main dnsmasq does
not fix that stale state.

After adding or changing a hostname in **Network → DHCP and DNS → Hostnames**:

1. Click **Save & Apply**.
2. SSH to Photonicat and send SIGHUP to the current Passwall dnsmasq PID. Do
   not hardcode the PID; it changes whenever Passwall restarts:
   ```sh
   passwall_dns_pid="$(
     ps w | awk '/\/dnsmasq_default / {print $1; exit}'
   )"
   test -n "$passwall_dns_pid" && kill -HUP "$passwall_dns_pid"
   ```
   SIGHUP clears that process's DNS cache and reloads its `addn-hosts` data
   without interrupting proxy connections.
3. Verify the actual port-53 path on Photonicat, then verify from a client:
   ```sh
   nslookup -type=A <name>.absurdlab.dev 127.0.0.1
   ```
   ```sh
   dig <name>.absurdlab.dev
   ```

If the targeted reload fails, restart Passwall as a fallback; this briefly
interrupts proxied connections:

```sh
/etc/init.d/passwall restart
```

## Troubleshoot Caddy DNS-01 behind Passwall

An ACME propagation timeout does not necessarily mean that the Cloudflare API
token or DNS update failed. A challenge record can be visible through a public
resolver while Caddy still receives an empty TXT answer through the local DNS
path. Without a certificate, HTTPS reaches Caddy but the TLS handshake fails
with an internal-error alert before the request reaches the backend.

Check the same active challenge from outside and inside the Caddy container:

```sh
dig +short TXT _acme-challenge.<name>.absurdlab.dev @1.1.1.1
```

```sh
docker exec caddy nslookup -type=TXT \
  _acme-challenge.<name>.absurdlab.dev 127.0.0.11
docker exec caddy nslookup -type=TXT \
  _acme-challenge.<name>.absurdlab.dev 1.1.1.1
```

Run the checks while Caddy logs `trying to solve challenge`; Caddy removes the
temporary TXT record after an attempt finishes. If the public query sees the
record but both container queries are empty, inspect Photonicat:

```sh
uci get 'passwall.@global[0].dns_redirect'
iptables-save -t nat 2>/dev/null | grep -E -- 'PSW_DNS|--dport 53'
```

With Passwall `dns_redirect=1`, its `PSW_DNS` chain redirects all LAN TCP and
UDP port-53 traffic to Passwall's private resolver. This includes packets
addressed explicitly to `1.1.1.1`, so Caddy's global `tls_resolvers` setting
cannot bypass the router-level redirect by itself. Adding resolver addresses
to Passwall's Direct List only changes proxy routing; it does not exempt those
queries from `PSW_DNS`.

Choose the DNS policy deliberately:

- Setting Passwall **DNS Redirect** to off allows explicitly addressed public
  DNS queries to escape interception. Clients using Photonicat as their DNS
  server still use its DNS path, but clients that hardcode port-53 resolvers
  can now bypass Passwall DNS enforcement.
- Keeping **DNS Redirect** on requires a persistent, source-specific firewall
  exemption for the Caddy host (`192.168.20.11`) and its explicit resolvers.
  A one-off rule is insufficient because Passwall rebuilds its chains.

Keep `tls_resolvers 1.1.1.1 1.0.0.1` in Caddy after either policy is applied,
then verify `certificate obtained successfully` in the Caddy logs and test the
served certificate from a client.

## Bootstrap on a fresh box

1. In Dockhand, deploy the `networks/proxy` stack (creates the `proxy` network).
2. Deploy `caddy` (set `CF_API_TOKEN` as a secret in its stack env-var panel
   first; let the custom image build once).
3. Deploy each backend stack.

## Why these choices

- #2 — Dockhand is a Compose-driven UI with manual Deploy; one repo backs many
  independent stacks; per-directory diff scoping.
- #4 — Cloudflare DNS-01; single SAN cert for `*.absurdlab.dev` + apex; LAN
  resolution via photonicat per-entry dnsmasq hijack.
- #8 — Dockhand-native secrets (Stack editor env-var panel, not Config sets);
  `.env.example` per stack; `restart: unless-stopped` with manual post-reboot
  redeploy; Dockhand Backup + off-host `ENCRYPTION_KEY` for recovery.
- #5 — this layout: `networks/proxy` bootstrap, `compose.yaml`, one name
  everywhere, mandatory `.env.example`, healthchecks if available.
