# Photonicat firmware-upgrade risk and Tailscale install path

> Resolves wayfinder ticket #02. Compiled 2026-08-07 via web research.
> Canonical identity: **Ariaboard Photonicat** (original V1 = Rockchip RK3568; Photonicat 2 = RK3576). Vendor: ariaboard.com / photonicat.com. OpenWRT wiki: [openwrt.org/toh/ariaboard/photonicat2](https://openwrt.org/toh/ariaboard/photonicat2) (covers Photonicat 2). The Photonicat is a niche product; official docs are thin, so general OpenWRT sysupgrade/Tailscale knowledge (which fully generalizes) carries most of the weight below.

## TL;DR for the reluctant user

- The dnsmasq/DNS-hijack and sidecar-gateway config **can be preserved** through an upgrade. OpenWRT sysupgrade saves/restores `/etc/config/` (where `dnsmasq`, `network`, `firewall`, `dhcp` live) by default when "keep settings" is used. This is not a coin-flip; it is the documented design.
- The bigger realistic risk is **losing the working setup to a bad flash**, not to config erasure. That risk is **low but non-zero** on an RK3568 board with eMMC, and it is **fully mitigated by a `sysupgrade -b` backup tarball + keeping the old image + the microSD fallback** (Rockchip boards boot from SD if inserted). You are not flying blind.
- Whether you need to upgrade **at all** depends on your current OpenWRT version, which you have not pinned down. Realistic ranges and the decision rule are below. If you are already on 23.05+, the reluctance is the only blocker.

## 1. Hardware and OpenWRT version

Two distinct products share the "Photonicat" name. The user almost certainly has the **V1 (RK3568)** given the sidecar-at-192.168.20.2 context and the pre-existing working setup, but both are covered.

### Photonicat V1 (original) — RK3568

- **SoC:** Rockchip RK3568 (4× Cortex-A55 @ 2.0 GHz, with NPU). [community/doc]
- **RAM:** 1 GB LPDDR4 (some variants 4 GB). [vendor/community]
- **Storage:** 8 GB eMMC + microSD slot. [vendor/community]
- **Ethernet:** 2× Gigabit (this is the physical reality of the sidecar — two ports). [vendor]
- **OpenWRT target:** `rockchip/armv8`. [openwrt wiki]
- **Shipped/supported OpenWRT:** Vendor firmware based on **OpenWRT 22.03.4** (~April 2023, kernel 6.1) is the documented historical baseline; V1 firmware images on `dl.photonicat.com` date from late 2022 / early 2023. The mainline OpenWRT device tree for "Ariaboard Photonicat RK3568" was submitted upstream in late 2024 / early 2025 (mailing-list patches), meaning mainline-stock support is recent. [doc/community]
- **LuCI:** yes, vendor images ship LuCI. [community/changelog]

### Photonicat 2 — RK3576

- **SoC:** Rockchip RK3576 (4× Cortex-A72 + 4× Cortex-A53, 6 TOPS NPU). [openwrt wiki]
- **RAM/Storage:** 4/16 GB LPDDR5, 64/128 GB eMMC. [openwrt wiki]
- **Current vendor firmware:** based on **OpenWRT 25.12**, kernel 6.12.x, Wi-Fi 7 support, built-in Docker. [vendor changelog]

### What version does Tailscale need?

The OpenWRT `tailscale` package requires **OpenWRT 23.05 or later** to work cleanly (doc-backed: the 22.03 package "is unable to configure nftables automatically" and breaks forwarding/daemon init unless you pass `--netfilter-mode=off`). On **22.03 (fw3/iptables)** Tailscale is usable only with manual workarounds; on **23.05+ (fw4/nftables)** it works out of the box. [openwrt wiki]

Storage note: devices with ≤16 MB flash cannot run the standard package. The Photonicat V1's **8 GB eMMC** makes this a non-issue — plenty of room. [openwrt wiki]

### Is the user's version actually too old?

**Cannot determine the exact current version from public sources** — this requires running `ubus call system board` on the device. The realistic decision rule:

- If on **22.03.x**: technically Tailscale installs but with the nftables workaround pain → **worth upgrading to 23.05+** for clean operation. Reluctance is partially justified (old fw3/nftables transition is exactly the friction zone).
- If on **23.05+**: Tailscale works cleanly. Reluctance is the **only** blocker.
- If on **a 2025+ vendor image / 24.10 / 25.x**: everything works; the package may just be stale (see §4).

## 2. Firmware upgrade procedure and brick risk

### Procedure

OpenWRT firmware is replaced by **sysupgrade**, either via LuCI (System → Backup / Flash Firmware) or CLI (`sysupgrade -v /tmp/image.bin`). sysupgrade wipes the OS partitions, installs the new image, and (if "keep settings" is on) restores the saved config files. [openwrt wiki]

For the Photonicat specifically, there are **three flash paths** and which one applies depends on whether you use vendor images or mainline:

1. **sysupgrade over the running system** (LuCI "Flash new firmware image" or `sysupgrade` CLI) — the normal in-place path. Works if you have a `*-sysupgrade.bin/.img.gz` built for this board.
2. **microSD reflash** — Rockchip boots from a valid SD card if inserted (`rk3568-photonicat-openwrt-sdboot-*.img.gz` style images). This is the clean "reinstall from outside the running OS" path.
3. **Maskrom/eMMC reflash via `rkdeveloptool`** from a PC — the full low-level recovery path, entered by holding the Maskrom button on power-on. This is also the **brick recovery** path. [openwrt wiki / vendor]

### Realistic brick risk

- **Low for a config-preserving sysupgrade**, because config lives in `/etc/config/` and is backed up/restored by design. The risk is a **bad upload / wrong image / power loss mid-flash**, not config loss.
- **The Photonicat's two safety nets are unusually strong**: (a) eMMC + separate microSD boot means an SD card image can always rescue a broken eMMC install; (b) Rockchip Maskrom mode + `rkdeveloptool` can reflash eMMC from a PC even if the device won't boot. This is **better recovery than most consumer routers** (which often have only a locked bootloader).
- Mitigations the OpenWRT docs themselves mandate: keep a backup, keep the **old** firmware image on hand, ideally keep a spare device. Have all three before starting. [openwrt wiki]

## 3. CRITICAL — does the upgrade wipe dnsmasq / DNS-hijack / sidecar-gateway?

**No, not if "keep settings" is used and your files are in the keep list.** This is the single most important fact for the reluctant user. [openwrt wiki — doc-backed]

### What sysupgrade preserves by default

sysupgrade's backup (`sysupgrade -b`) and the "keep settings" flag save and restore the standard OS config, **including `/etc/config/`** — which is exactly where the homelab backbone lives:

- `/etc/config/dnsmasq` — the DNS-hijack rules for `*.absurdlab.dev`
- `/etc/config/network` — the sidecar-gateway at 192.168.20.2, its upstream next-hop
- `/etc/config/firewall` — zones/forwarding for the sidecar
- `/etc/config/dhcp` — DHCP behavior

These are all under `/etc/config/` and are preserved by the default keep list. [openwrt wiki — doc-backed]

### The keep list: `/etc/sysupgrade.conf`

Anything **outside** the default-preserved directories must be added to `/etc/sysupgrade.conf` or it is lost. The OpenWRT Tailscale guide itself demonstrates this pattern (e.g. `echo '/etc/hotplug.d/iface/90-ethtool' >> /etc/sysupgrade.conf`). For the homelab, audit:

- Any custom dnsmasq conf files in `/etc/dnsmasq.d/` or `/tmp/...` — add to sysupgrade.conf.
- Any init/hotplug scripts the DNS-hijack depends on — add to sysupgrade.conf.
- Custom `/etc/hosts` entries if used for hijacking — `/etc/hosts` is preserved, but verify.

Known caveat (community-sourced, doc-adjacent): [openwrt/openwrt#17913](https://github.com/openwrt/openwrt/issues/17913) reports `/etc/sysupgrade.conf` entries not always honored on some major-version jumps (e.g. 23.05.5 → 24.10). Treat the keep list as best-effort and **always have the full tarball as the source of truth**, not the in-place keep list alone.

### Recommended backup/restore sequence

1. **Full config tarball** before anything: `sysupgrade -b` → produces `/tmp/sysupgrade.tar.gz` (or via LuCI: System → Backup / Flash Firmware → Generate archive). Download it off-device. **This is the single file that contains the entire homelab backbone.**
2. **Explicit copies** of the four config files as plain text (`/etc/config/{dnsmasq,network,firewall,dhcp}`), plus any custom dnsmasq includes. Belt-and-suspenders.
3. **Keep the current firmware image** on your PC for emergency reflash.
4. After upgrade, if config didn't auto-restore: LuCI → Backup / Flash Firmware → "Restore backup" and upload the tarball; or manually re-drop the four files and `/etc/init.d/network restart; /etc/init.d/dnsmasq restart`.
5. **Verify the hijack** before declaring victory: from a client on 192.168.20.0/24, `dig foo.absurdlab.dev` should resolve to the private IP, not a public DNS answer.

The homelab backbone (dnsmasq hijack + sidecar gateway) is **purely config, in preserved locations** — there is no kernel-module or binary dependency. That makes it one of the most upgrade-resilient pieces of the whole setup.

## 4. Installing Tailscale on OpenWRT

### Package and feed

- Official package name: **`tailscale`**, dependency **`kmod-tun`**. [openwrt wiki — doc-backed]
- On 23.05: `opkg update && opkg install tailscale`. On 24.10+ (and snapshots): the package manager is **`apk`** (`apk add tailscale`). [openwrt wiki — doc-backed]
- **Version reality (important):** the `tailscale` package in the OpenWRT repo is **chronically stale / behind upstream**, often stuck around 1.5x while upstream moves to 1.7x+. Community consensus (reddit, GitHub issues) is to not rely on the repo package for current versions. [community-sourced]
- The OpenWRT wiki itself says: "Depending on your OpenWrt version the package included may be outdated and missing security updates," and points to update instructions. [doc-backed]

### Getting a current Tailscale on this device

Three realistic paths, in increasing effort:

1. **Repo package** (`opkg install tailscale`) — easiest, may be stale.
2. **Community feed** (e.g. lanrat.com's openwrt-tailscale-repository) — backports current Tailscale as opkg packages. Unofficial. [community-sourced]
3. **Manual ARM64 binary** — drop the upstream `tailscale`+`tailscaled` aarch64 build over the opkg install (nwgat.ninja documents this for FriendlyWRT/ARM64; directly applicable since RK3568 is `aarch64`). [community-sourced]

Architecture note: RK3568 is **`aarch64_cortex-a55`** — standard, well-supported by all Tailscale ARM64 builds.

### Does the package survive sysupgrade?

- **Config/state survives** if you add `/etc/tailscale/` (the state dir with login/keys) to `/etc/sysupgrade.conf` — without this you re-auth after every upgrade. [doc-backed pattern]
- **The package binary does NOT survive a standard sysupgrade** — only files in the keep list are restored; installed packages are wiped and must be reinstalled (`opkg install tailscale` again, or restore your manual binary). This is OpenWRT's general behavior, confirmed across multiple sources. [doc/community]
- **Attended Sysupgrade (ASU)** is the clean fix: it rebuilds the image **with your currently-installed package list baked in**, so Tailscale reinstalls automatically and configs restore from the keep list. ASU is included in stock OpenWRT by default and works for stock-to-stock transitions. **Caveat for the Photonicat:** ASU is designed for stock/mainline OpenWRT; on **vendor firmware** it may not be available or may not work (the Photonicat V1 vendor image is built from a forked SDK). If you stay on vendor firmware, plan on manual package reinstall after each upgrade. [doc-backed, with vendor caveat]

### Vendor-firmware special case (Photonicat)

A recurring community finding across vendor-OpenWRT devices (GL.iNet, and by extension Photonicat-style boards): **firmware upgrades overwrite the Tailscale version with whatever the firmware image bundles.** So even a "successful" vendor-firmware upgrade can downgrade your carefully-installed current Tailscale back to whatever the image ships. Expect to reinstall/re-upgrade Tailscale after any vendor firmware update. [community-sourced]

## 5. Subnet routing and exit node on OpenWRT

**Both supported.** [openwrt wiki — doc-backed]

### Subnet router

```
tailscale up --advertise-routes=192.168.20.0/24 --snat-subnet-routes=false
```

- `--snat-subnet-routes=false` is the OpenWRT-specific recommendation: OpenWRT does its own masquerading via firewall zones, so disable Tailscale's SNAT to preserve real source IPs. [doc-backed]
- Approve the route in the Tailscale admin console after.

### Exit node

```
tailscale up --advertise-exit-node ...
```

- Approved in admin console, then it's a WAN gateway for tailnet peers. [doc-backed]
- Also needs the OpenWRT firewall to forward `tailscale0` → WAN zone. The OpenWRT wiki covers adding the interface and firewall zone for `--advertise-exit-node`.

### OpenWRT-version caveats (the big one)

- **23.05+ = fw4/nftables** → Tailscale configures firewall rules automatically. Clean.
- **22.03 = fw3/iptables** → the 22.03 Tailscale package "is unable to configure nftables automatically," breaking forwarding and daemon init. Workaround: run with `--netfilter-mode=off` and hand-roll rules. **This is the single strongest reason to be on 23.05+ before depending on Tailscale.** [doc-backed]

### Other caveats

- **IPv6 exit node:** default-route config can restrict to LAN prefix only; disable IPv6 Source Routing in WAN6 to work around. [doc-backed]
- **Throughput (24.10 + Tailscale 1.54+, kernel 6.6):** install `ethtool`, enable `rx-udp-gro-forwarding` on the WAN interface for meaningfully higher WireGuard throughput. [doc-backed]
- **Kernel modules:** `kmod-tun` required (pulled as a dependency). RK3568 mainline kernel support is recent (DTS patches landed late 2024/early 2025); on older vendor kernels all needed modules ship. [doc/community]

## 6. Exit node exits through the Photonicat's own proxy/routing — for free

**Confirmed to hold, with one nuance.** [architectural reasoning, doc-adjacent]

- An exit node advertises the device's default route to the tailnet. The Photonicat is the sidecar gateway at 192.168.20.2; its own default route / next-hop is the UDM/upstream (per the topology). Traffic that enters via `tailscale0` destined for "the internet" follows the Photonicat's normal forwarding path → out its next-hop → through whatever proxy/routing the Photonicat already imposes.
- Therefore: yes, a tailnet peer using the Photonicat as exit node **egresses through the Photonicat's existing proxy path at no extra configuration cost**. The same plumbing the sidecar-gateway clients already use carries exit-node traffic. This is the "single ideal box" property the ticket calls out.
- **Nuance / things to verify:**
  1. **Masquerading on the Photonicat's upstream interface** must cover the Tailscale client source range. If the Photonicat already masquerades its LAN (192.168.20.0/24) toward the UDM, you must extend that masquerade (or the firewall rule) to also cover `100.64.0.0/10` (the Tailscale CGNAT range) — otherwise return traffic from the UDM to 100.x.x.x will be dropped/blackholed. This is the most common "exit node works from the router itself but not from peers" failure mode on OpenWRT.
  2. **`net.ipv4.ip_forward`** must be 1 (it is, since the Photonicat already routes for its clients) and forwarding must be permitted `tailscale0` → WAN in the fw4 ruleset.
  3. The DNS authority property stacks cleanly: the Photonicat can remain the DNS server for tailnet peers too, so the `*.absurdlab.dev` hijack extends over the tunnel if you advertise the Photonicat as the DNS resolver (Tailscale's `--accept-dns` on clients).

The exit-node-through-proxy property is **free in design** but the masquerade-for-100.64.0.0/10 check is the one operational item to confirm on first bring-up.

## Confidence notes

- **Doc-backed:** Tailscale package name/feed (`tailscale` + `kmod-tun`), 23.05+ recommendation, 22.03 nftables breakage + `--netfilter-mode=off` workaround, subnet-router/exit-node flags, `--snat-subnet-routes=false`, `/etc/sysupgrade.conf` keep-list pattern, sysupgrade preserving `/etc/config/`, ASU behavior (rebuilds with installed package list), `sysupgrade -b` tarball, 16 MB flash limit (N/A here), Photonicat 2 hardware (RK3576), OpenWRT target rockchip/armv8.
- **Community-sourced:** Photonicat V1 = RK3568 / 1 GB / 8 GB eMMC specs (vendor page is sparse; corroborated by AndroidPimp and mailing-list DTS), V1 firmware on OpenWRT 22.03.4 baseline (Telegram/mailing-list), OpenWRT repo `tailscale` package being chronically stale, vendor-firmware upgrades overwriting the bundled Tailscale version (GL.iNet parallel, applied to Photonicat by analogy), ASU not being reliable on vendor firmware.
- **Undetermined from public docs:** the user's exact current OpenWRT version (must run `ubus call system board`), whether the Photonicat V1 vendor image bundles Tailscale at all (no evidence it does — assume not), whether ASU works on the Photonicat V1 vendor image (assume not; plan for manual reinstall), the exact Photonicat V1 firmware upgrade mechanics for crossing the 22.03 → 23.05 boundary on vendor images (the vendor SDK is forked; the mainline DTS only landed in late 2024/early 2025), whether the OpenWRT wiki `photonicat` (V1) page will ever populate (currently empty).

## Sources

[openwrt.org/toh/ariaboard/photonicat2](https://openwrt.org/toh/ariaboard/photonicat2) (Photonicat 2 hardware, target rockchip/armv8, install/upgrade methods) · [photonicat.com/photonicat_v1](https://photonicat.com/photonicat_v1) (V1 vendor specs) · [dl.photonicat.com/_README_.html](https://dl.photonicat.com/_README_.html) (V1 firmware image listing) · [photonicat.com/wiki — Photonicat 2 firmware changelog](https://photonicat.com/wiki/Photonicat_2_%E5%9B%BA%E4%BB%B6_Changelog) (vendor firmware on OpenWRT 25.12) · [ariaboard.com/wiki](https://ariaboard.com/wiki/Main_Page) (RK3568 SDK / build guide) · [github.com/ariaboard-com/rockchip_rk3568_linux_sdk (openwrt branch)](https://github.com/ariaboard-com/rockchip_rk3568_linux_sdk) (V1 source) · [linux-rockchip mailing list — Add support for Ariaboard Photonicat RK3568](https://lists.openwrt.org/pipermail/linux-rockchip/2025-January/054569.html) (upstream DTS, Jan 2025) · [androidpimp.com — Photonicat router](https://www.androidpimp.com/embedded/photonicat-router/) (V1 specs corroboration) · [openwrt.org — Tailscale](https://openwrt.org/docs/guide-user/services/vpn/tailscale/start) (package, 23.05 req, nftables, subnet router, exit node, sysupgrade.conf) · [openwrt.org — generic.sysupgrade](https://openwrt.org/docs/guide-user/installation/generic.sysupgrade) (sysupgrade behavior, keep list, brick prep) · [openwrt.org — techref/sysupgrade](https://openwrt.org/docs/techref/sysupgrade) (`/etc/sysupgrade.conf`, `/lib/upgrade/keep.d/*`) · [openwrt.org — attended.sysupgrade](https://openwrt.org/docs/guide-user/installation/attended.sysupgrade) (ASU rebuilds with installed packages) · [github.com/openwrt/openwrt#17913](https://github.com/openwrt/openwrt/issues/17913) (sysupgrade.conf not always honored on big jumps) · [superuser.com — packages on sysupgrade](https://superuser.com/questions/277704/what-happens-to-installed-packages-on-a-sysupgrade-in-openwrt) (non-default packages wiped) · [tailscale.com/docs — Exit Nodes](https://tailscale.com/kb/1103/exit-nodes) · [tailscale.com/docs — Subnet Routers](https://tailscale.com/kb/1019/subnets) · [lanrat.com — openwrt-tailscale-repository](https://lanrat.com/projects/openwrt-tailscale-repository/) (community feed) · [nwgat.ninja — Tailscale ARM64 on OpenWRT](https://nwgat.ninja/install-or-upgrade-tailscale-on-openwrt-arm64-aarch64/) (manual binary) · [reddit r/Tailscale — update on OpenWRT](https://www.reddit.com/r/Tailscale/comments/1fqzqmy/how_do_you_update_tailscale_on_openwrt/) (repo package staleness) · [github.com/tailscale/tailscale#10856](https://github.com/tailscale/tailscale/issues/10856) (upgrade difficulties, vendor vs stock) · [forum.gl-inet — update Tailscale on all devices](https://forum.gl-inet.com/t/script-update-tailscale-on-nearly-all-devices/37582) (vendor-firmware overwrites Tailscale) · [wundertech.net — Tailscale on OpenWrt](https://www.wundertech.net/how-to-set-up-tailscale-on-openwrt/) (subnet/exit-node walkthrough)
