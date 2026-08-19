# UDM Pro Tailscale installer briefing

> Resolves wayfinder ticket #01. Compiled 2026-08-07 via web/GitHub research.
> Canonical project: **`SierraSoftworks/tailscale-unifi`** (formerly *tailscale-udm*), [github.com/SierraSoftworks/tailscale-unifi](https://github.com/SierraSoftworks/tailscale-unifi).

## The big surprise: the old project is gone

The classic installer the ticket assumed was canonical — `unifi-utilities/tailscale-on-udm` — **no longer exists.** `gh api repos/unifi-utilities/tailscale-on-udm` returns **404** as of 2026-08-07. The whole `unifios-utilities` org was archived into `unifi-utilities/unifios-utilities-archived`, and the surviving `unifi-utilities` org now ships shared libraries only (`unifi-common`, `unifi-common-addons`) — no first-party Tailscale installer.

The **de-facto installer is now `SierraSoftworks/tailscale-unifi`** (MIT, ~1.66k stars, topics include `udm-pro`). It is the project every 2025–2026 guide points at. SierraSoftworks is a third party (not Ubiquiti, not Tailscale).

## Maturity / current state (all doc/API-backed)

- **Latest release:** v3.3.4, published **2026-07-22**. Releases are frequent: v3.3.3 (2026-06-21), v3.3.2 (2026-06-21), v3.3.1 (2026-06-13), v3.3.0 (2026-05-27).
- **Last push:** 2026-07-28; last repo activity 2026-08-06. **Actively maintained.**
- **Open issues:** 1 (a UCG Max performance report, #205). Issue tracker is well-triaged; recent issues close fast.
- **Install method:** pulls **Tailscale's official apt package** from `pkgs.tailscale.com` (on UniFi OS 2.x+, which is Debian/bullseye-based). Not a static binary blob. So the Tailscale binary itself is upstream-supported; only the *packaging glue* is SierraSoftworks.
- **Compatibility:** UniFi OS 2.x and later, including **UDM Pro** explicitly. The README's v3.x line dropped UniFi OS 1.x (legacy podman) — those devices must use v2.8.0 on the `legacy` branch.

## How it survives firmware updates and reboots (the central question)

This is the part the ticket is really asking about, and the answer changed substantially in 2026. Three layers of defence, designed against the fact that **UniFi OS firmware updates wipe `/usr` (binaries) but preserve `/data` (= `/ssd1/.data` on UDM-SE/UDM-Pro) and the overlay-root `/etc`.**

1. **State persistence.** `tailscaled.state` lives at `/data/tailscale/tailscaled.state`. The Tailscale login/identity survives an update because `/data` is not wiped. Confirmed empirically in issue #186: after a 5.1.12 update, "the UDM-SE [came] back online … same `tailscaled.state` identity preserved."
2. **A `tailscale-install.service` + `.timer` reinstall the apt package.** The service is `Type=oneshot`, `ExecStart=/data/tailscale/manage.sh on-boot`, ordered `After=network-online.target`, with `Restart=on-failure` / `RestartSec=5m` and `StartLimitIntervalSec=0` (retries indefinitely). The timer fires `OnBootSec=2m` and `OnUnitActiveSec=24h`. Comment in the unit file: *"the primary mechanism for keeping Tailscale installed across firmware updates is the systemd tailscale-install.service/.timer."*
3. **The unit files themselves are the persistence bootstrap**, and this is the fragile bit that broke and got fixed in 2026.

**Known breakage modes (chronological, all issue-backed):**

- **#149 / #150 — UniFi OS 5.0.x major-version bump broke install.** Updating to 5.0.10–5.0.12 produced "Unsupported UniFi OS version (v5)" until users re-ran the (updated) installer. **Pattern: every UniFi OS major bump historically needs an installer update; users who don't update the installer first lose Tailscale until they SSH in and re-run `curl … install.sh | sh`.**
- **#186 — `/ssd1` mount race on UniFi OS 5.1.12 (2026-05-22).** This is the most instructive failure. On UDM-SE, `/data` → `/ssd1/.data`, and `/ssd1` mounts late in boot. The v3.2.0 installer had placed unit files in `/data/tailscale/` and symlinked them from `/etc/systemd/system/`. At early boot systemd evaluated those symlinks before `/ssd1` was mounted → "Failed to open /ssd1/.data/tailscale/tailscale-install.timer" → timer never armed → **exit node offline, no auto-recovery, manual reinstall required.** `RequiresMountsFor=` inside the unit could not help because the unit itself was never loaded.
  - **Fix (v3.3.0, 2026-05-27; hardened v3.3.3, 2026-06-21):** unit files are now copied as **plain regular files into `/etc/systemd/system/`** (on the always-mounted overlay root), not symlinks into `/data`. The `ExecStart` still points at `/data/tailscale/manage.sh`, where `RequiresMountsFor=/data/tailscale` now correctly applies because the unit is loaded. The current `package/tailscale-install.service` and `.timer` reflect this layout. A reboot test in #186 passed cleanly post-fix.
  - **Upgrade gotcha (community-reported, since-handled):** `cp -f` doesn't replace existing symlinks, so upgrading from the old symlink-based layout required `rm -f` of the old symlinks first (or `cp --remove-destination`). Current `manage.sh` does the `-L`/remove dance; a fresh v3.3.4 install avoids the problem entirely.
- **DNS-not-ready race (community-reported in #186, orthogonal):** `tailscale-install.service` can briefly fail to resolve `pkgs.tailscale.com` if binaries were wiped and DNS isn't ready at boot. The service handles it gracefully (`Restart=on-failure` retries every 5 min) and the daily timer is the backstop. Not a correctness bug, but explains occasional "didn't come back for a few minutes after an update" reports.
- **#118 — "didn't auto-`up` after upgrade" (2025).** Maintainer's position: auto-start with no manual `tailscale up` is expected; if it doesn't, check `journalctl -eu tailscaled`. In practice the saved state should bring the node up automatically; needing a manual `up` indicates a startup failure, not normal behaviour.

**Recovery steps when it breaks (distilled from the issues):**
```sh
curl -sSLq https://raw.githubusercontent.com/SierraSoftworks/tailscale-unifi/main/install.sh | sh
/data/tailscale/manage.sh install!      # force reinstall, repairs unit files
systemctl status tailscale-install.timer # confirm "active (waiting)"
tailscale status                         # identity should be intact from /data
```
If unit-file symlinks are stuck from an old layout: `rm -f /etc/systemd/system/tailscale-install.{service,timer}` then `manage.sh install!`.

## Subnet router and exit node: both yes

Both are fully supported and **documented in the README** (not just community claims). Two important specifics:

- **Requires TUN mode, not userspace networking.** Modern installs default to TUN. Verify with `ip link show tailscale0`; if missing, you're on legacy userspace mode and must edit `/data/tailscale/tailscale-env` to remove `--tun userspace-networking` and run `manage.sh install`.
- **Subnet router + exit node together (README "Final Configuration" recipe):**
  ```sh
  tailscale up --advertise-exit-node \
    --advertise-routes="10.0.0.0/24,192.168.20.0/24" \
    --snat-subnet-routes=false --accept-routes --reset
  ```
  This is exactly the form the ticket needs: advertise `192.168.20.0/24` (the Photonicat VLAN) and any other VLANs, and offer exit-node at the same time. `net.ipv4.ip_forward` is enabled by default on UniFi OS.

**Critical interaction (community-sourced, issue #173):** `--snat-subnet-routes=false` (the README's recommended subnet-router setting) **is incompatible with using the node as an exit node.** Reporter smasher816 traced this to [tailscale/tailscale#18725](https://github.com/tailscale/tailscale/issues/18725): with `NoSNAT=true`, the gateway loses internet when `--exit-node` is set; setting `--snat-subnet-routes=true` fixes it. **So you cannot use the README's exact combined recipe for an exit node that also does per-subnet split routing** — see the next section. This is a real, load-bearing constraint for the Photonicat topology.

## KEY QUESTION — routing exit-node traffic through the Photonicat at 192.168.20.2

This is the deciding factor for the ticket, and the honest answer is: **it is architecturally awkward, requires hand-rolled Linux policy routing that Tailscale's own maintainers decline to support in the GUI/CLI, and it has only been demonstrated by one community member on a *different* topology.** It has not been demonstrated for your exact "downstream-next-hop-gateway" shape.

**First, clarify what "exit node" means here.** Tailscale's `--exit-node` points at a *remote* tailnet node that egresses to *its own* internet. There is no Tailscale primitive for "send my exit-node traffic to a LAN next-hop gateway (192.168.20.2) for local egress." So the goal must be restated as one of:

**(A) UDM is the Tailscale exit node, and you want its exit traffic to leave via the Photonicat instead of the UDM's own WAN.**
This is a pure Linux policy-routing problem on the UDM, downstream of Tailscale. It is *in principle* solvable — you'd mark Tailscale-decapsulated packets (fwmark `0x80000/0xff0000` is what tailscaled uses internally) and `ip rule` them to a next hop of `192.168.20.2` instead of the default route. But:

- Tailscale on the UDM already installs its own fwmark rules (table 52, the `0x80000` mark, the `main`/WAN lookups) precisely to push encapsulated WireGuard traffic out the WAN. Re-pointing that at a LAN gateway means fighting tailscaled's own route table, in the same fragile way issue #173 fights it for per-VLAN split tunneling.
- **Nobody has published a working recipe for "Tailscale exit node on UDM, egress via a downstream LAN router."** The closest published work is issue #173, which does the *opposite* (per-VLAN → remote Tailscale exit node), and even *that* required ~40 lines of bespoke `ip rule`/`ip route` commands, manual table-52 surgery, and a `systemd` unit to re-apply it on every boot and to re-remove tailscaled's default route whenever it re-appears. The reporter's own words: *"a workable but hacky solution,"* with a `monitor_routes` loop that deletes the default route whenever tailscaled re-adds it.

**(B) Photonicat is the Tailscale exit node; the UDM is "just" a subnet router for 192.168.20.0/24 and other VLANs.**
This is the architecturally clean option and the one the evidence supports. Put Tailscale on **both** boxes: the UDM advertises all VLANs as subnet routes (the only box that can cleanly see every VLAN), and the Photonicat runs Tailscale as the exit node. Remote clients pick the Photonicat as exit node. Then exit traffic *naturally* lands on the Photonicat and egresses there — no UDM policy routing, no fighting tailscaled's route table. The Photonicat is already OpenWRT, where `tailscale` is a first-class opkg package with a LuCI app, no installer-jank.

**Recommendation implied by the evidence:** Option B. The UDM's job is subnet routing (its unique capability); the Photonicat's job is exit-node egress (its existing role as the proxy gateway). Concretely: `tailscale up --advertise-routes=…` on the UDM with `--snat-subnet-routes=false`; `tailscale up --advertise-exit-node` on the Photonicat. This sidesteps every fragility in section above and matches how the Photonicat is already wired (clients on `192.168.20.0/24` already use it as gateway and it already hijacks `*.absurdlab.dev` DNS via dnsmasq).

**Why option A is fragile even if you attempt it:** UniFi OS firmware updates reset networking/routing state in ways the installer doesn't control (it only guarantees the Tailscale *package* and *state* survive). Any custom `ip rule` table you add on the UDM for next-hop redirection is **not** in `/data`, will be **wiped by firmware updates**, and must be re-applied by a boot hook you write and maintain yourself — the exact "fragile, hand-maintained" outcome the ticket wants to avoid. Combined with the documented `snat-subnet-routes=false` ↔ exit-node conflict, option A is a multi-front maintenance burden on a box whose OS updates have already broken this installer twice in 2026.

## Backup / restore

**There is no built-in backup/restore feature.** `manage.sh` has no backup, dump, tar, or restore code path (verified by reading the source). The only data-management commands are `install`, `update`, `uninstall` (which is just `apt remove -y tailscale` + deleting the systemd units), `cert`.

The de-facto backup story is implicit and split:

- **What survives on its own (no action needed):** `tailscaled.state` (login identity, node key) and `tailscale-env` (your flags) under `/data/tailscale/`. Because `/data` is not wiped by updates, your Tailscale identity is effectively self-backing.
- **What does NOT survive a full wipe (e.g. factory reset, SSD failure):** the systemd unit files in `/etc/systemd/system/` and the apt source. These are re-derived by re-running `install.sh`, so "restore" == "reinstall." There is no exported bundle.
- **Practical backup recipe (community-pattern, not project-documented):** periodically `tar` `/data/tailscale/` (state + env + manage.sh) to off-box storage. On a wiped device, restore that dir then run `install.sh`; the installer will rebuild `/etc` units and reinstall the apt package, and the restored `tailscaled.state` brings back the same node identity. This is robust because the installer is idempotent and state is just a file — but you have to build and schedule the tar yourself.
- **If an update wipes everything and you have no backup:** re-run `install.sh` and `tailscale up` again. You get a working install but a **new node identity** (old node shows "stale" in the admin console until removed); advertised routes and exit-node approvals must be re-accepted in the Tailscale ACL/admin UI. Annoying, not catastrophic.

## Bottom line for host selection (informs blocked ticket #03)

- The installer is real, current (v3.3.4, 2026-07-22), and actively maintained, and its persistence model is now well-engineered against the `/ssd1` mount race that bit it in May 2026. **As a subnet router on the UDM, it is a sound choice** — and the UDM is the only box that can cleanly advertise every VLAN.
- **As the exit node whose traffic must traverse the Photonicat, it is the wrong choice.** That requires unsupported, update-fragile, hand-rolled policy routing that fights tailscaled's own route tables, with no published recipe for the downstream-next-hop shape. The Photonicat should be the exit node.
- Residual risk to plan for regardless: **every UniFi OS major-version bump is a maintenance event.** Budget for "SSH in, re-run installer, re-confirm timer armed" roughly once per UniFi OS major release (the 2026 track record: 5.0.x broke it, 5.1.x broke it differently). The daily timer plus `/data` state means outages are recoverable in minutes, not hours, but they are not zero-touch.

## Confidence notes

- **Doc/source-backed:** project identity (404 of old repo, SierraSoftworks as current de-facto installer), version v3.3.4 / 2026-07-22, MIT, UDM-Pro support, apt-based install, TUN-mode requirement, subnet-router + exit-node README recipe, the `tailscale-install.service`/`.timer` persistence mechanism, `/data/tailscale/tailscaled.state` location, absence of any backup/restore feature in `manage.sh`, the README's `--snat-subnet-routes=false` recommendation.
- **Issue/issue-comment-backed:** the `/ssd1` mount-race breakage and its v3.3.0/v3.3.3 fix (#186); UniFi OS 5.0.x breaking the installer until re-run (#149, #150); identity surviving across updates (#186); the `snat-subnet-routes=false` ↔ exit-node conflict (#173, citing tailscale/tailscale#18725); the per-VLAN split-exit-node policy-routing recipe and its "hacky" self-assessment (#173); the expected-auto-start-after-upgrade behaviour (#118).
- **Community-sourced (single reporter, treat as directional not proven):** the full working `ip rule`/`ip route`/systemd recipe in #173 is from one user (smasher816) on a UCG-Fiber, peer-reviewed only by a maintainer saying "LGTM" — it has not been reproduced at scale or on a UDM Pro specifically.
- **Undetermined from public sources:** whether anyone has run the *downstream-next-hop-gateway* variant (UDM exit node egressing via a LAN router like 192.168.20.2). No public recipe exists; the conclusion that it is "architecturally awkward and unsupported" is an inference from the absence of a recipe plus the documented difficulty of the simpler per-VLAN case, not a direct statement by Tailscale or SierraSoftworks. Whether the next UniFi OS update breaks the v3.3.x persistence model again is also unknown until it ships.

## Sources

[github.com/SierraSoftworks/tailscale-unifi](https://github.com/SierraSoftworks/tailscale-unifi) · [SierraSoftworks/tailscale-unifi README](https://github.com/SierraSoftworks/tailscale-unifi/blob/main/README.md) · [package/tailscale-install.service](https://github.com/SierraSoftworks/tailscale-unifi/blob/main/package/tailscale-install.service) · [package/tailscale-install.timer](https://github.com/SierraSoftworks/tailscale-unifi/blob/main/package/tailscale-install.timer) · [package/on-boot.sh](https://github.com/SierraSoftworks/tailscale-unifi/blob/main/package/on-boot.sh) · [package/manage.sh](https://github.com/SierraSoftworks/tailscale-unifi/blob/main/package/manage.sh) · [release v3.3.0](https://github.com/SierraSoftworks/tailscale-unifi/releases/tag/v3.3.0) · [#173 per-VLAN split exit node + policy routing recipe](https://github.com/SierraSoftworks/tailscale-unifi/issues/173) · [#186 /ssd1 mount race on UniFi OS 5.1.12 and v3.3.x fix](https://github.com/SierraSoftworks/tailscale-unifi/issues/186) · [#149 UniFi OS 5.0.x unsupported](https://github.com/SierraSoftworks/tailscale-unifi/issues/149) · [#150 UniFi OS 5 unsupported](https://github.com/SierraSoftworks/tailscale-unifi/issues/150) · [#118 auto-start after firmware upgrade](https://github.com/SierraSoftworks/tailscale-unifi/issues/118) · [tailscale/tailscale#18725 snat-subnet-routes vs exit node](https://github.com/tailscale/tailscale/issues/18725) · [unifi-utilities/unifios-utilities-archived](https://github.com/unifi-utilities/unifios-utilities-archived) · [tailscale exit-nodes docs](https://tailscale.com/kb/1103/exit-nodes) · [tailscale subnet-router docs](https://tailscale.com/kb/1019/subnets)
