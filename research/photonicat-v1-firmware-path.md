# Photonicat V1 firmware path: 21.02 → Tailscale-capable, with sciencenet preservation

> Resolves wayfinder ticket #04. Compiled 2026-08-08 via web research (Photonicat wiki + dl.photonicat.com image listings + community repos).
> Device in scope: **Ariaboard Photonicat V1** (Rockchip RK3568, 1 GB LPDDR4, 8 GB eMMC + microSD, 2× GigE), confirmed via SSH banner running **OpenWRT 21.02, r3786-ab06308b8**.
> This file sharpens `research/photonicat-firmware-upgrade.md` (ticket #02) with two facts that earlier file lacked: (1) the now-confirmed 21.02 starting point and the real V1 firmware train that extends to 2025, and (2) the actual identity of **sciencenet** and its reinstall story. The earlier file remains the authoritative reference for the general sysupgrade/Tailscale/nftables/exit-node theory and is not duplicated below; only the deltas and the ticket's five specific questions are answered here.

## TL;DR for the deciding user

- **Yes, a clean 21.02 → 23.05+ path exists for the V1, and it is vendor-supported.** The V1 is NOT abandoned. Photonicat keeps publishing `photonicatwrt-*` sysupgrade images for the RK3568 board; the latest is **`photonicatwrt-25.02.0-r7541`** (2025-06-04 changelog entry), kernel **6.12**, synced with upstream OpenWRT (effectively OpenWRT 24.10-class — well above the 23.05 floor Tailscale needs). [doc-backed]
- **The recommended target is `photonicatwrt-25.02.0` (the R25.02 train).** It is the current stable V1 line, kernel 6.12 gives clean fw4/nftables for Tailscale, and the V1's sciencenet ipk packages are explicitly built for this kernel series. [doc-backed for the image; doc-backed for the nftables floor]
- **Sciencenet is a separately-installed ipk bundle, NOT part of the firmware.** It will be **wiped by any reflash** and must be **reinstalled** after the upgrade. Its **config survives** if it lives in `/etc/config/` and you keep the `sysupgrade -b` tarball. The reinstall path is the same one the user used originally: download `science_net.tar.xz` from `openwrtkitty/photonicat-ipks`, upload at `http://172.16.0.1/update`. [doc-backed install procedure; doc-adjacent for the wipe/restore mechanics — same OpenWRT behavior the #02 file already established]
- **The decisive mitigation for bricking fear** is unchanged from #02 but now concrete for the V1: the Rockchip Maskrom + `rkdeveloptool`/RKDevTool recovery path (the vendor's own documented V1 flash method) can reflash eMMC from a PC even if the box won't boot. Combined with a config tarball + the old image kept on hand, the homelab backbone is recoverable. [doc-backed]
- **Net: the topology gate (#03) clears.** The Photonicat V1 can run Tailscale cleanly post-upgrade. Proceed with the upgrade, not the host-behind-Photonicat pivot.

## What is NEW here vs photonicat-firmware-upgrade.md

| Topic | #02 said | #04 (this file) sharpens to |
|---|---|---|
| Current version | "Cannot determine — run `ubus call system board`" | **Confirmed: OpenWRT 21.02, r3786-ab06308b8** (SSH banner). Below the 22.03 friction zone. |
| Latest V1 firmware | "V1 images on dl.photonicat.com date from late 2022 / early 2023" — **WRONG / stale.** | V1 receives ongoing `photonicatwrt-*` builds through **2025-06-04**. Latest stable train is **R25.02** (kernel 6.12, upstream-synced). V1 is actively maintained. |
| V1 OpenWRT base | Assumed ~22.03.4 historical baseline | R25.02 = OpenWRT 24.10-class (kernel 6.12). The floor for Tailscale (23.05+) is met and exceeded. |
| V1 upgrade procedure | Listed sysupgrade / SD / Maskrom generically, vendor-caveat | **Vendor-documented V1 method is RKDevTool USB Maskrom burn** for crossing versions; **SD-card path only works for firmware ≤ 2023-01-30**; in-place sysupgrade over LuCI is mechanically possible (the `*-sysupgrade.img.gz` naming) but is NOT the vendor's documented cross-version path. See §2. |
| "sciencenet" | Not covered — was an unknown | **Identified and resolved.** See §4. It is `science_net.tar.xz`, the `openwrtkitty/photonicat-ipks` bundle of PassWall/OpenClash/SSR+/OpenVPN/ZeroTier. |
| Recommended plugins | Not surveyed | Surveyed in §5. |

## 1. Can I upgrade at all? Is the V1 still supported?

**Yes, and emphatically so.** The V1 is still actively shipped firmware by the vendor. [doc-backed]

- The download server at `dl.photonicat.com/images/` hosts a continuous train of V1 OpenWRT images. The naming switched from `BOARDCONFIG-RK3568-PHOTONICAT-OPENWRT-…` (2022–early 2023, raw eMMC burn images) to **`photonicatwrt-<YY.MM>-r<rev>-<hash>-rockchip-armv8-ariaboard_photonicat-squashfs-sysupgrade.img.gz`** starting 2023-04-20 (R23.04). The `ariaboard_photonicat` target name = V1 hardware. [doc-backed, directory listing of dl.photonicat.com/images/ and /images/beta/]
- The **latest V1 sysupgrade image** is `photonicatwrt-25.02.0-r7541-61dbcce7d-rockchip-armv8-ariaboard_photonicat-squashfs-sysupgrade.img.gz` (released across 2025, changelog final entry 2025-06-04). [doc-backed]
- The vendor's [Photonicat 1 firmware changelog](https://photonicat.com/wiki/Photonicat_1_%E5%9B%BA%E4%BB%B6_Changelog) confirms active development through 2025: R23.04 → R23.07 → R24.04 → **R25.02**, with 2025-06-04 adding ZeroTier and SMB4 sharing on top of R25.02. [doc-backed]
- The V1 wiki section is large and current (Photonicat Specs, Flashing Manual, SD Boot/Update, OpenWRT Plugin Install, OpenWRT Firmware Update Guide, OTA Update, FAQs, Compile OpenWRT/Debian, TurboACC, etc.). Development has **not** moved entirely to the Photonicat 2 — the V1 is a maintained first-class product. [doc-backed]

**Conclusion on Q1:** A published V1 firmware image far newer than 21.02 exists. The latest is photonicatwrt-25.02 (kernel 6.12, upstream-synced). The V1 is vendor-supported, not orphaned.

## 2. How do I perform the upgrade? (V1-specific procedure)

There are three V1 paths. Which one applies depends on (a) the size of the version jump and (b) whether you go vendor-image → vendor-image or cross to a different OS. **The vendor's documented cross-version path for the V1 is the USB Maskrom burn, not LuCI sysupgrade.** This is the single most important correction to the assumptions in #02. [doc-backed]

### Path A — RKDevTool USB Maskrom burn (vendor-documented V1 update method)

This is what the [Photonicat OpenWrt Firmware Update Guide](https://photonicat.com/wiki/Photonicat_OpenWrt%E5%9B%BA%E4%BB%B6%E6%9B%B4%E6%96%B0%E6%8C%87%E5%8D%97) actually describes. It is the path the vendor expects you to use to go from an old 21.02 build to a 2025 build. [doc-backed]

1. Download from `dl.photonicat.com/images/`:
   - The firmware: the latest `photonicatwrt-25.02.0-r7541-…-sysupgrade.img.gz` (despite the "sysupgrade" in the name, it is used as the burn image here).
   - `MiniLoaderAll.bin` (Rockchip bootloader loader, also in the same directory).
   - The [RKDevTool v2.86](https://photonicat.com/wiki/Photonicat_%E5%88%B7%E6%9C%BA%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8C) Windows flash tool + its USB driver.
2. **Decompress the `.img.gz`** — the guide is explicit: "固件必须解压，否则烧录不成功" (the firmware MUST be extracted or flashing fails).
3. **Enter Maskrom mode on the Photonicat**: short-press the button 3×, then hold 15 s, until LED1 blinks rapidly. **Then** connect the A-to-A USB cable to the PC. (Connecting before entering Maskrom means the tool won't see the device.)
4. In RKDevTool: "Download Image" tab → select `MiniLoaderAll.bin` as the loader → select the extracted firmware → Execute. Wait 1–2 min for "下载完成" (download complete) and a solid LED, then manually reboot.
5. Caveats from the guide: must use kernel-6.1.24+ firmware for this method (everything R23.04 onward qualifies; R25.02 is fine). Uninstall Huorong antivirus on the PC (it blocks Maskrom detection). Try a different USB cable if not detected.

This **wipes eMMC and reinstalls** — settings are NOT auto-preserved here. The config-tarball restore step (§3 of #02) is mandatory after a Path-A flash.

#### 4.19 → 6.x specific notes (confirmed against the user's box)

The user's confirmed kernel string is **`4.19.232-photonicat #1 SMP Mon Jan 30 17:15:31 CST 2023`** — dated exactly on the 2023-01-30 SD-card boundary. This is the boundary case the wiki's date language refers to. Verified against the [Photonicat 刷机操作手册](https://photonicat.com/wiki/Photonicat_%E5%88%B7%E6%9C%BA%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8C) (flashing manual). [doc-backed]

- **No intermediate step.** Crossing 4.19 → 6.x is a single Maskrom burn, not two hops. There is no required "first reach a 6.x baseline, then to 25.02" intermediate firmware. The burn completely overwrites the system including the bootloader, so there is no "4.19 bootloader trying to boot a 6.12 kernel" mismatch.
- **The 4.19 vs 6.x distinction is procedural, not path-related.** Two procedural differences when the *source* is a 4.19-era build:
  - **Erase flash first (mandatory).** The manual: "USB线刷不同主线内核系统时请先擦除flash" (when USB-flashing across different mainline kernel systems, erase the flash first). RKDevTool has an EraseFlash step you run before the burn. This handles the implied partition/layout change between 4.19-era and 6.x-era images.
  - **Use the latest RKDevTool + manually select `MiniLoaderAll.bin`.** Older 4.19-era flashing used a one-click upgrade flow; 6.x requires manually selecting the bootloader loader file first, then the firmware image.
- **The SD-card path is dead for this purpose, confirmed.** The user's firmware is the last version the SD method could write, but the SD method cannot write a 6.x target. Maskrom USB is the only path to 25.02.
- **What gets written:** both the bootloader (uboot/idbloader via `MiniLoaderAll.bin`) and the OS image. This is why the kernel transition is safe in one hop, the full stack from bootloader up is replaced.

**Exact procedure for this box (4.19.232, 2023-01-30 → R25.02):** (1) `sysupgrade -b` backup + copy off `/etc/config/{dnsmasq,network,firewall,dhcp}` and the sciencenet config (`/etc/config/passwall` or `/etc/config/shadowsocksr` or `/etc/openclash/`) + export subscription URLs/node lists as plain text; (2) enter Maskrom mode; (3) **EraseFlash** in RKDevTool; (4) burn `MiniLoaderAll.bin` + `photonicatwrt-25.02.0-r7541-…-sysupgrade.img.gz`; (5) on first boot, restore the config tarball via LuCI (or re-drop the files and `restart` the services); (6) reinstall sciencenet at `http://172.16.0.1/update`; (7) verify the DNS hijack and sidecar gateway before declaring victory.

### Path B — LuCI / CLI sysupgrade (in-place, same-train upgrades)

The `*-sysupgrade.img.gz` filename on the V1 images means the squashfs-sysupgrade image is **mechanically valid** for OpenWRT's sysupgrade flow (LuCI → System → Backup / Flash Firmware → upload, keep-settings ticked; or `sysupgrade /tmp/image.bin`). This is the normal path for **minor in-train upgrades** (e.g. R24.04 r6322 → R24.04 r6338, or a patch within R25.02). [doc-adjacent — the image format is sysupgrade-capable; the vendor docs just don't headline it for big jumps]

**Whether sysupgrade cleanly carries 21.02 → 25.02 in one hop is undetermined from public docs.** No source explicitly demonstrates this jump on a Photonicat V1. OpenWRT's general guidance (and #02's coverage) is that very large version gaps risk config-format incompatibilities — the safe posture for a jump this big is to **treat it as a full reflash (Path A), restore the config tarball, reinstall sciencenet**. Plan for Path A; Path B is a bonus if it works. [doc-adjacent / reasoning]

### Path C — SD-card boot / sdupdate

The [SD Boot & Update](https://photonicat.com/wiki/Photonicat_SD%E5%90%AF%E5%8A%A8%E3%80%81%E6%9B%B4%E6%96%B0) page documents that uboot probes for a valid bootable SD card and boots from it if found, and the `sdupdate` image rewrites eMMC from SD. **Crucially: "SD卡刷仅适用于2023年1月30日及之前的固件版本"** — SD-card flashing only applies to firmware ≤ 2023-01-30. That means **the SD path is NOT usable for reaching R25.02.** It's a recovery option for old images only. Use `dd`/Win32 Disk Imager to write `rk3568-photonicat-openwrt-sdboot-*.img.gz` to the card. [doc-backed]

### Recovery (brick insurance)

- Maskrom mode + RKDevTool/rkdeveloptool reflashes eMMC from a PC **even if the device will not boot**. This is the V1's hard recovery floor and the main reason the bricking fear is overblown — you can always get back to a working state with a USB cable and the image file. [doc-backed, vendor flashing manual + TTL/Watchdog/Recovery page]
- The 2023-01-30 SD-card floor is worth keeping one old `sdboot` image around for, as a second-tier rescue, but it is not the primary recovery path for R25.02. [doc-backed]

## 3. What version number should I upgrade to?

**Recommended target: `photonicatwrt-25.02.0-r7541-61dbcce7d-rockchip-armv8-ariaboard_photonicat-squashfs-sysupgrade.img.gz`** (the latest R25.02 build on `dl.photonicat.com/images/`). [doc-backed]

Rationale:

- **Meets the Tailscale floor with margin.** R25.02 runs kernel 6.12 and is upstream-synced, i.e. OpenWRT 24.10-class. fw4/nftables is the default firewall — Tailscale configures its rules automatically with no `--netfilter-mode=off` workaround. The 23.05 floor from #02 is comfortably exceeded. [doc-backed for the kernel/version; doc-backed for the Tailscale-on-fw4 behavior from #02]
- **It is the current vendor-supported line.** Going R25.02 keeps you on the patch-receiving train; older trains (R24.04, R23.x) are frozen at their last build date. [doc-backed]
- **Sciencenet ipks target this kernel series.** The `openwrtkitty/photonicat-ipks` release notes explicitly call out compatibility with "Linux 6.1.82, openWRT R24.04" and newer; the R25.02 (6.12) line is within the supported envelope. Going older does not help; going bleeding-edge third-party (see below) risks ipk ABI mismatch. [doc-backed for the ipk release notes; doc-adjacent for the kernel-match reasoning]

### Alternatives considered (not recommended for this user)

- **Mainline OpenWRT snapshot for `ariaboard_photonicat`.** The Photonicat V1 device tree was upstreamed to mainline OpenWRT in early 2025 ([lists.openwrt.org patch series](https://lists.openwrt.org/pipermail/linux-rockchip/2025-January/054599.html)). A mainline snapshot would give the cleanest, most generic OpenWRT — but it would **drop `pcat-manager`** (the vendor's Python web UI for the battery, modem, LCD, MCU — the things that make the Photonicat a Photonicat and not a generic RK3568 board), and mainline snapshots are a moving target. Recommended only if the user wants to abandon the vendor UI entirely. Not the right call when the goal is a low-risk Tailscale upgrade that preserves sciencenet. [doc-backed for the upstreaming; reasoning for the rest]
- **Third-party QWRT / ZagLu immortalwrt.** The wiki's Main Page links these as alternatives (`github.com/coolsnowwolf/lede`, `pcat.qsim.top/zagwrt/immortalwrt`). They often bundle proxy plugins out of the box and may have better sciencenet integration, but they are community builds with their own bug surface. **Cross-firmware-type transitions require USB Maskrom burn (cannot sysupgrade)** and you'd be revalidating the whole stack from scratch. Reject for now; revisit only if vendor R25.02 proves unsuitable. [doc-backed for existence and the cross-type burn requirement; community-sourced for quality]
- **R24.04 (last build 2024-12-25).** Functional and above the 23.05 floor, but strictly older than R25.02 with no upside. Only choose if R25.02 surfaces a regression. [doc-backed]

## 4. CRITICAL — sciencenet: what it is, and the reinstall/reconfigure story

This is the deciding question for the user's bricking fear. Answered definitively below.

### What sciencenet actually is

**"sciencenet" is the user's local label for the Photonicat community's `science_net.tar.xz` package** — the **科学上网** ("scientific internet access", i.e. proxy/circumvention) bundle from the [`openwrtkitty/photonicat-ipks`](https://github.com/openwrtkitty/photonicat-ipks) GitHub repo, purpose-built for the Photonicat. It is **NOT a generic OpenWRT package** in the official feeds, and **NOT bundled into the firmware image**. It is a separately-installed ipk/tarball. [doc-backed]

`science_net.tar.xz` is a tarball of multiple `.ipk` files that bundles, per the repo README and release notes: **PassWall, Clash, OpenClash, SSR+ (ShadowsocksR Plus+), OpenVPN, and ZeroTier.** The user's "sciencenet" proxy is one or more of these (most likely PassWall or OpenClash, the two dominant choices in the Chinese-language Photonicat community). [doc-backed]

The same repo also ships `shairplay.tar.xz`, `transmission.tar.xz`, `ddnsto`, UU Game Booster, and `adguardhome` — confirming this is the de-facto Photonicat plugin distribution channel. [doc-backed]

### Install / reinstall procedure (the path the user will re-walk)

Documented at [Photonicat 安装openWRT插件](https://photonicat.com/wiki/Photonicat_%E5%AE%89%E3%A3%85openWRT%E6%8F%92%E4%BB%B6): [doc-backed]

1. Download `science_net.tar.xz` from the repo's Releases page (latest release tagged `r7720`, with explicit compatibility notes for "Linux 6.1.82, openWRT R24.04" and newer — R25.02 qualifies).
2. Browse to **`http://172.16.0.1/update`** (the Photonicat's `pcat-manager-web` update page — note this is `pcat-manager`, not LuCI; LuCI itself is at `http://172.16.0.1:8080`).
3. Drag `science_net.tar.xz` into the upload box → click Upload.
4. Wait for the Log window to show "Finished". Some plugins need a reboot.
5. Verify in OpenWRT (LuCI at :8080) → Services — the proxy plugins appear there.

### Does it survive a firmware upgrade? (the real question)

**No, the package does not survive. Yes, the config does (with the right prep).** This is OpenWRT's general sysupgrade behavior, applied to the V1: [doc-backed mechanics, established in #02 §4 and re-confirmed]

- **The package binaries are wiped by any reflash.** Whether you go Path A (Maskrom burn) or Path B (sysupgrade), non-preinstalled packages like PassWall/OpenClash do not come back on their own. The firmware image does not bundle them. **Expect to reinstall sciencenet after the upgrade.** [doc-backed]
- **The plugin config survives if it is in `/etc/config/` and you preserve it.** PassWall lives at `/etc/config/passwall`, OpenClash at `/etc/openclash/`, SSR+ at `/etc/config/shadowsocksr`. These are UCI config files in standard locations.
  - Under **Path B (sysupgrade with "keep settings")**: these are preserved by default (`/etc/config/` is in the keep list). Add `/etc/openclash/` to `/etc/sysupgrade.conf` if using OpenClash (it's outside `/etc/config/`).
  - Under **Path A (Maskrom burn)**: nothing is preserved automatically. You **must** take a `sysupgrade -b` tarball first (LuCI → System → Backup / Flash Firmware → Generate archive, or `sysupgrade -b` over SSH → downloads `/tmp/sysupgrade.tar.gz`), and **restore it after the flash** (LuCI → Backup → Restore backup, upload the tarball). [doc-backed, OpenWRT generic; the V1 adds no caveat]
- **The single most important pre-step** — before touching the firmware: SSH in, run `sysupgrade -b`, copy `/tmp/sysupgrade.tar.gz` off the box, AND manually copy the proxy config files (`/etc/config/passwall`, `/etc/config/shadowsocksr`, `/etc/openclash/`) and any subscription URLs / node lists to your PC as plain-text belt-and-suspenders. The community consensus (Chinese forum threads) is explicit here: **manually export subscription links / node lists as a separate backup**, because config formats are not always backward-compatible across plugin versions. [community-sourced; doc-adjacent]

### The bricking-fear bottom line

The homelab backbone the user is afraid of losing has **two recoverable halves**:

1. **The dnsmasq `*.absurdlab.dev` hijack + sidecar-gateway at 192.168.20.2** — pure `/etc/config/{dnsmasq,network,firewall,dhcp}` config, preserved by sysupgrade, restorable from tarball. Already covered in #02 §3. The most upgrade-resilient piece.
2. **The sciencenet proxy itself** — binaries wiped (reinstall via `172.16.0.1/update` in 5 min), config preserved/restored from tarball. The reinstall is a known, documented, ~5-minute path, not an archaeological dig.

Combined with Maskrom recovery as the hard floor, **there is no realistic failure mode that loses the sciencenet setup permanently**, as long as the tarball + node-list backups are taken first. The fear is the right fear to have (it forces the backup discipline), but the answer is "back up, then the path is safe", not "don't upgrade". [doc-backed for the mechanics; reasoning for the conclusion]

## 5. Recommended plugins / packages for this hardware

Surveyed from the Photonicat wiki's scenario pages + the `openwrtkitty/photonicat-ipks` repo + OpenWRT ecosystem. Ranked, with relevance flags.

### Tier 1 — directly relevant to the Tailscale/proxy/DNS goals (install these)

| Package | What it does | Why for this effort | Source |
|---|---|---|---|
| **`science_net.tar.xz`** (PassWall and/or OpenClash) | The proxy. This **is** sciencenet. | Mandatory reinstall — it's the homelab backbone. Choose PassWall (simpler) or OpenClash (Mihomo-based, more flexible rule engine). Note: OpenClash's first start often needs another proxy to fetch its kernel — bring the box up on PassWall first if you go OpenClash. | [doc-backed](https://github.com/openwrtkitty/photonicat-ipks) |
| **`tailscale` + `kmod-tun`** | The whole point of the upgrade. | Required for the sidecar-as-exit-node topology. On R25.02 (fw4/nftables) it configures cleanly. | [doc-backed](https://openwrt.org/docs/guide-user/services/vpn/tailscale/start); see #02 §4 for version-staleness and the manual-binary fallback |
| **`luci-app-tailscale`** (asvow build) | LuCI GUI for Tailscale (start/stop, up/down, status, login). | The official OpenWRT `tailscale` pkg has no UI; this fills the gap. Install the ipk manually from the asvow repo. | [community-sourced](https://github.com/asvow/luci-app-tailscale) |
| **`ethtool`** | NIC feature toggles. | Needed for the WireGuard-throughput trick (`rx-udp-gro-forwarding` on the WAN iface) on kernel 6.6+. Real Tailscale throughput win. | doc-backed; see #02 §5 |
| `luci-app-firewall` (fw4) | nftables UI. | Already in the vendor image; verify present. Needed to add the `tailscale0` → WAN forwarding rule for the exit node. | doc-backed |

### Tier 2 — general-purpose nice-to-haves from the Photonicat wiki

| Package | What it does | Why / when | Source |
|---|---|---|---|
| `adguardhome` | Network-wide DNS ad blocking. | Useful if the Photonicat is the DNS authority for the tailnet (which, per #02 §6, it can be). Conflicts mildly with dnsmasq on port 53 — plan the split. | [openwrtkitty/photonicat-ipks](https://github.com/openwrtkitty/photonicat-ipks) |
| `transmission` | BitTorrent. | Only if the Photonicat's storage is used for downloads. Not relevant to Tailscale goals. | same repo |
| `shairplay` | AirPlay receiver. | Music Box mode (wiki has a page). Pure nice-to-have. | same repo |
| `ddnsto` | Remote access / NAT-traversal remote console. | Overlaps with Tailscale's job; redundant in this topology. Skip unless you want vendor-style OOB access. | same repo |
| TurboACC / `turboacc` (flow offload) | Hardware flow offloading. | Wiki has a dedicated page. Can help throughput, but Tailscale's WireGuard userspace path doesn't benefit from kernel offload the same way — test before committing. | [doc-backed](https://photonicat.com/wiki/Photonicat_%E7%BD%91%E7%BB%9C%E5%8A%A0%E9%80%9F_Turbo_ACC) |
| Samba4 | SMB sharing. | R25.02 already bundles SMB4 per the 2025-06-04 changelog. No extra install needed. | [doc-backed, changelog](https://photonicat.com/wiki/Photonicat_1_%E5%9B%BA%E4%BB%B6_Changelog) |
| ZeroTier | Mesh VPN. | Bundled inside `science_net.tar.xz`. Choose **Tailscale OR ZeroTier**, not both — they fight for similar turf. Tailscale wins for this topology (exit-node feature, tailnet DNS). | [doc-backed](https://github.com/openwrtkitty/photonicat-ipks) |

### Final ranked recommendation for the upgrade day

1. `science_net.tar.xz` → install whichever of PassWall/OpenClash you were using before (don't switch engines mid-upgrade). Restore its config from the tarball.
2. `opkg update && opkg install tailscale kmod-tun` (or `apk add` if R25.02's apk-fed). Add `/etc/tailscale/` to `/etc/sysupgrade.conf` so login state survives future upgrades.
3. `luci-app-tailscale` (asvow ipk) for the UI.
4. `ethtool` + enable `rx-udp-gro-forwarding` on WAN once Tailscale is verified working.
5. Defer everything in Tier 2 until the backbone (DNS hijack + sidecar gateway + Tailscale exit node) is verified end-to-end.

## Confidence notes

- **Doc-backed (high confidence):** the V1 is actively supported; the firmware train extends to `photonicatwrt-25.02.0-r7541` (2025-06-04); R25.02 uses kernel 6.12 and is upstream-synced (OpenWRT 24.10-class); the V1 vendor-documented update method is RKDevTool USB Maskrom burn via `MiniLoaderAll.bin`; SD-card flashing is limited to firmware ≤ 2023-01-30; **sciencenet = `science_net.tar.xz` = the openwrtkitty/photonicat-ipks bundle of PassWall/OpenClash/SSR+/OpenVPN/ZeroTier**; plugins install via `http://172.16.0.1/update`; the R25.02 firmware does not bundle proxy plugins (the build's included list is pcat-manager / pcat-manager-web / quectel-cm / modem+power management — no proxy); Maskrom is the brick-recovery path; package binaries do not survive reflash; config in `/etc/config/` survives sysupgrade's keep list; mainline Photonicat V1 DTS upstreamed early 2025.
- **Community-sourced (medium-high):** the `luci-app-tailscale` asvow build being the de-facto LuCI UI; OpenClash needing a working net connection to fetch its kernel on first start (so bring up PassWall first); the recommendation to manually export proxy node lists / subscription URLs as a separate backup; QWRT/ZagLu third-party builds being usable but bug-prone and requiring cross-type USB burn.
- **Undetermined from public docs:**
  - Whether **LuCI/CLI sysupgrade cleanly carries 21.02 → 25.02 in one hop** on a V1. No source demonstrates it. Safe posture: assume a full reflash (Path A) is needed; treat Path B as a bonus if it works.
  - The **exact upstream OpenWRT commit/base** R25.02 is synced to (the changelog says "synced with upstream" without naming the release; kernel 6.12 strongly implies OpenWRT 24.10-class, but the vendor SDK's `openwrt` submodule points to a forked commit `8a1e3fa` that does not exist in the mainline openwrt repo — i.e. it's a vendor fork, not a clean mainline tag). The 23.05+ floor for Tailscale is met regardless.
  - Whether the user's current "sciencenet" is PassWall or OpenClash specifically — needs a look at `/etc/config/` on the device to confirm (`ls /etc/config/ | grep -Ei 'passwall|shadowsocksr|openclash'`).
  - The wiki's English coverage of the V1 update procedure (the English wiki is thinner than the Chinese `wiki.photonicat.cn` mirror; some pages 500 intermittently but the Chinese originals load and were used for this research).

## Sources

[photonicat.com/wiki/Main_Page](https://photonicat.com/wiki/Main_Page) (wiki index; V1 and V2 page listings, third-party firmware links) · [photonicat.com/wiki/Photonicat_1_固件_Changelog](https://photonicat.com/wiki/Photonicat_1_%E5%9B%BA%E4%BB%B6_Changelog) (V1 firmware changelog through 2025-06-04, R23.04→R25.02) · [photonicat.com/wiki/Photonicat_OpenWrt固件更新指南](https://photonicat.com/wiki/Photonicat_OpenWrt%E5%9B%BA%E4%BB%B6%E6%9B%B4%E6%96%B0%E6%8C%87%E5%8D%97) (V1 update = RKDevTool USB Maskrom burn procedure) · [photonicat.com/wiki/Photonicat_刷机操作手册](https://photonicat.com/wiki/Photonicat_%E5%88%B7%E6%9C%BA%E6%93%8D%E4%BD%9C%E6%89%8B%E5%86%8C) (Maskrom entry: 3× short-press + 15 s hold, LED1 fast-blink) · [photonicat.com/wiki/Photonicat_SD启动、更新](https://photonicat.com/wiki/Photonicat_SD%E5%90%AF%E5%8A%A8%E3%80%81%E6%9B%B4%E6%96%B0) (SD-card flashing limited to firmware ≤ 2023-01-30) · [photonicat.com/wiki/Photonicat_安装openWRT插件](https://photonicat.com/wiki/Photonicat_%E5%AE%89%E8%A3%85openWRT%E6%8F%92%E4%BB%B6) (plugin install via `http://172.16.0.1/update`) · [photonicat.com/wiki/Photonicat_编译_OpenWRT,_Debian](https://photonicat.com/wiki/Photonicat_%E7%BC%96%E8%AF%91_OpenWRT,_Debian) (vendor image packages = pcat-manager / pcat-manager-web / quectel-cm; no proxy plugins) · [photonicat.com/wiki/Photonicat_网络加速_TurboACC](https://photonicat.com/wiki/Photonicat_%E7%BD%91%E7%BB%9C%E5%8A%A0%E9%80%9F_Turbo_ACC) · [photonicat.com/downloads](https://photonicat.com/downloads) (V1 ships with OpenWRT; images on dl.photonicat.com) · [dl.photonicat.com/images/](https://dl.photonicat.com/images/) (V1 stable listing incl. `photonicatwrt-25.02.0-r7541-…-sysupgrade.img.gz`, `MiniLoaderAll.bin`) · [dl.photonicat.com/images/beta/](https://dl.photonicat.com/images/beta/) (V1 beta listing, R23.04 → R24.04 train) · [github.com/openwrtkitty/photonicat-ipks](https://github.com/openwrtkitty/photonicat-ipks) and [/releases](https://github.com/openwrtkitty/photonicat-ipks/releases) (the sciencenet bundle: `science_net.tar.xz` = PassWall/Clash/OpenClash/SSR+/OpenVPN/ZeroTier; release r7720 compatible with R24.04+/kernel 6.1.82+) · [github.com/ariaboard-com/rockchip_rk3568_linux_sdk (openwrt branch)](https://github.com/ariaboard-com/rockchip_rk3568_linux_sdk) (V1 vendor SDK; `openwrt` submodule at forked commit `8a1e3fa`) · [lists.openwrt.org — Add support for Ariaboard Photonicat RK3568 (Jan 2025)](https://lists.openwrt.org/pipermail/linux-rockchip/2025-January/054599.html) (mainline DTS upstreaming) · [openwrt.org/toh/ariaboard/photonicat2](https://openwrt.org/toh/ariaboard/photonicat2) (target `rockchip/armv8`, install/upgrade methods) · [openwrt.org — Tailscale](https://openwrt.org/docs/guide-user/services/vpn/tailscale/start) (23.05+ requirement, nftables, subnet router, exit node) · [github.com/asvow/luci-app-tailscale](https://github.com/asvow/luci-app-tailscale) (community LuCI UI for Tailscale) · [club.photonicat.cn](https://club.photonicat.cn/) (unofficial forum; thread /d/186 on factory + third-party firmware and plugin sources) · [wiki.photonicat.cn](https://wiki.photonicat.cn/) (Chinese mirror of the wiki, used where the English pages intermittently 500) · Earlier file `research/photonicat-firmware-upgrade.md` (ticket #02) — carries the general sysupgrade/Tailscale/exit-node theory this file builds on.
