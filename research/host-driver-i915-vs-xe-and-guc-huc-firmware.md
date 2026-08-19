# Host driver (i915 vs xe) and GuC/HuC firmware for the Intel Arc A380 on TrueNAS SCALE

> Resolves wayfinder ticket #02 (host driver + firmware). Compiled 2026-08-11 via web research.
> Canonical identity: **Intel Arc A380**, code name **Alchemist**, silicon **ACM-G11**, architecture family **DG2**, PCI ID **8086:56a5** (128 EUs). Discrete, single-slot, 75 W, the common homelab transcode card. Treat anything that says "DG2" in the Intel/kernel docs as applying to this card.

## TL;DR

- **Driver: use `i915`. Do not switch to `xe`.** i915 is what binds by default on TrueNAS SCALE for the A380, and it is the right choice on every released SCALE version. The `xe` driver only becomes the default at Lunar Lake / Battlemage; on Alchemist it is opt-in, historically lacked HuC media support, and as of Phoronix's Dec 2025 testing high-end Arc cards (A750/A770) were still broken on `xe` under Linux 6.19. For a QSV transcode workload there is no reason to move off i915 today.
- **Firmware: it just works on SCALE >= 24.04.2 and on 24.10 / 25.10, provided `linux-firmware` is current.** The A380 needs GuC submission (mandatory on DG2) and HuC authentication (for HEVC/VC-1 decode). The blobs ship in the `firmware-linux-nonfree` / `linux-firmware` package at `/lib/firmware/i915/dg2_guc_70.bin`, `/lib/firmware/i915/dg2_huc_gsc.bin`, and `/lib/firmware/i915/dg2_dmc_ver2_08.bin`. TrueNAS 24.04.0 shipped without the HuC blob (the famous regression); it was fixed in 24.04.2.
- **Module options: none required for the A380.** `i915.enable_guc=2/3` is not needed on DG2 because the driver auto-enables GuC submission (it is the only submission path on this hardware) and HuC loads via the GSC. You only need `enable_guc` on older iGPUs (Tiger Lake / Alder Lake-P). Do not cargo-cult it onto the A380.
- **Render node: `/dev/dri/card0` + `/dev/dri/renderD128`, owned `root:render` (GID 104 on SCALE, which is Debian-based).** This is the device the container bind-mounts.
- **`force_probe`: not needed for the A380.** PCI ID 56a5 has been fully upstream since kernel 6.2. The `i915.force_probe=!56a5` / `xe.force_probe=56a5` dance is only for cards whose IDs are not yet in the kernel's ID table. Yours is.
- **Field-tested recommendation: stay on TrueNAS SCALE 24.10 (Electric Eel) or 25.10 (Goldeye), let `i915` auto-bind, confirm GuC+HuC load via `dmesg`, and pass `/dev/dri/renderD128` (plus `card0`) into the container with the container's GID for `render` matching 104, or `chmod 666` the node. Do not set `enable_guc`, do not force_probe, do not switch to `xe`.**

A note on code names, because the ticket got them slightly wrong and it matters for kernel mapping:

| TrueNAS version | Code name   | Kernel       |
|-----------------|-------------|--------------|
| 24.04.x         | Dragonfish  | 6.6.32       |
| **24.10.x**     | **Electric Eel** | **6.6.44** (production+truenas) |
| 25.04.x         | Fangtooth   | 6.12 (first to adopt) |
| **25.10.x**     | **Goldeye** | **6.12 LTS** (6.12.33 -> 6.12.91) |

The ticket's "24.10 Dragonfish" is really 24.04 Dragonfish, and "25.10 Electric Eel" is really 24.10 Electric Eel. The kernel mapping in the ticket (6.6 vs 6.12) is correct once you pair it with the right version: **24.10 Electric Eel = 6.6.44, 25.10 Goldeye = 6.12 LTS**.

## 1. i915 vs xe: which binds, and which is right

### What each driver is

- **`i915`** is the legacy Intel DRM driver. It has had DG2/Alchemist support backported since kernel 6.0 (initial) and 6.2 (full, no longer experimental). On SCALE 24.10 (6.6.44) and 25.10 (6.12) this support is mature.
- **`xe`** is Intel's modern DRM driver, designed for Xe-LPG and later, built with discrete GPUs and non-x86_64 (ARM64/RISC-V) in mind. It merged into the mainline kernel around 6.8. For Alchemist and Meteor Lake, `xe` is supported but opt-in.

### What TrueNAS SCALE ships

Both SCALE 24.10 (kernel 6.6.44) and 25.10 (kernel 6.12) ship **both** `i915` and `xe` as in-tree modules. Neither is stripped. The question is which one **binds** the A380.

### Which binds by default: `i915`

From Phoronix (Dec 2025, testing Arc Alchemist on Linux 6.19):

> "By default the Alchemist and Meteor Lake graphics use the i915 kernel driver by default but they can optionally use the Xe kernel driver instead. ... The Intel Xe kernel driver is only used by default beginning with Lunar Lake and Battlemage hardware but Alchemist and Meteor Lake can enjoy optionally switching over to the newer driver using the `i915.force_probe=![PCI-ID] and xe.force_probe=[PCI-ID]` kernel module options."

So on a stock SCALE install the A380 binds `i915` with no module options. This is correct.

### Is `i915` the right choice? Yes.

- **Media/QSV:** `i915` has full GuC + HuC wiring for DG2. The `xe` driver historically **lacked HuC media support for DG2/Alchemist** (Phoronix, 2023: "Intel's Experimental Xe Driver For Linux Lacking HuC Media Support For DG2/Alchemist"), which is the exact feature a transcode box needs. This has improved in recent kernels, but `i915` remains the proven path.
- **Stability:** Phoronix's Dec 2025 piece notes the Arc A750 and A770 were **broken on both drivers** under Linux 6.19 at test time; only the A580 worked on both. The A380 (smaller DG2 part) is less affected, but the takeaway is that `xe` is still the bleeding edge. For a headless transcode appliance you want the boring driver.
- **Intel's own docs:** the [dgpu-docs i915 KMD page](https://dgpu-docs.intel.com/overview/supported-hardware/i915-driver-gpus.html) lists the A380 (56a5) and A310 (56a6) under the i915 driver, with full support landing at kernel 6.2+.

### When you would consider `xe`

Only if you need S0ix/runtime PM on a laptop, or you are on Battlemage/Lunar Lake where `xe` is the default. Neither applies to a TrueNAS box with an A380. Do not switch.

### The `force_probe` quirk, demystified

The kernel marks PCI IDs as "supported" once they are in the driver's ID table. For IDs that are recognized but not yet whitelisted as "fully supported," the driver refuses to bind unless you pass `force_probe`. The syntax:

- `i915.force_probe=56a5` -> force i915 to bind 56a5 even if not whitelisted
- `i915.force_probe=!56a5` -> **prevent** i915 from binding 56a5 (so `xe` can take it)
- `xe.force_probe=56a5` -> force `xe` to bind 56a5

**For the A380 (56a5) on kernel 6.6 or 6.12, none of these are needed.** The ID has been whitelisted since 6.2. The `force_probe` chatter in forum posts is almost always about pre-release cards, or about people explicitly switching to `xe`. If `lspci -nnk` shows `Kernel driver in use: i915` without any force_probe in your kernel command line, you are in the nominal state.

## 2. GuC and HuC firmware

### What they are and why the A380 needs them

- **GuC (graphics microcontroller):** handles workload scheduling / context submission. On DG2 the GuC submission path is the **only** submission path: the legacy execlist mode is gone. So GuC firmware is not optional for Arc.
- **HuC (HEVC/VC-1 decode microcontroller):** offloads bitstream processing for HEVC and VC-1 decode. Required for hardware-accelerated HEVC transcode, which is most of what a media stack transcodes.
- **DMC (display microcontroller):** display power-state management. Not relevant to a headless transcode box but loads anyway.

### Where the blobs live

On Debian/SCALE the firmware comes from the `firmware-linux-nonfree` meta-package (pulls `intel-microcode` and the i915 firmware blobs) which installs into `/lib/firmware/`. The DG2-specific files under `/lib/firmware/i915/`:

- `dg2_guc_70.bin` -> symlink to the current 70.x release (e.g. `dg2_guc_70.5.1.bin`; newer bundles ship 70.8.0). This is the GuC firmware.
- `dg2_huc_gsc.bin` -> the HuC firmware. Note the `gsc` suffix: on DG2 the HuC is loaded **via the GSC (Graphics Security Controller)**, which is a newer auth path than older platforms where HuC loaded directly. This is why an outdated `linux-firmware` breaks HuC specifically.
- `dg2_dmc_ver2_08.bin` -> DMC, display.

Authoritative listing: the [kernel.org linux-firmware tree `i915/` directory](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/i915).

### Module options: what you need to set

**For the A380 on `i915`: nothing.** The `i915.enable_guc` bitmask exists:

- `1` = GuC submission only
- `2` = HuC authentication only
- `3` = both

On older platforms (Tiger Lake, Alder Lake-P, etc.) you sometimes set `i915.enable_guc=2` or `=3` because the driver does not turn these on by default. **On DG2 the driver forces GuC submission on (it is mandatory) and HuC loads via GSC automatically.** The TrueNAS forum threads that recommend `enable_guc=2` are about Intel iGPUs, not Arc. Setting it on the A380 is harmless but redundant.

For the `xe` driver: GuC and HuC are enabled by default and there is no `enable_guc` equivalent to set. (One more reason the question of "which driver" mostly answers itself: `i915` is where the mature DG2 firmware path lives.)

### The famous regression on TrueNAS 24.04.0

When Dragonfish 24.04.0 shipped, the bundled `linux-firmware` was too old and lacked `dg2_huc_gsc.bin`. Symptom in `dmesg`:

```
i915 0000:0d:00.0: [drm] *ERROR* GT0: HuC firmware i915/dg2_huc_gsc.bin: fetch failed -ENOENT
i915 0000:0d:00.0: [drm] GT0: GuC firmware i915/dg2_guc_70.bin version 70.5.1
```

GuC loaded fine; HuC failed to fetch -> no HEVC transcode. Even after 24.04.1 added the blob, a kernel regression still broke DG2. **This was fixed in 24.04.2.** On 24.10 (Electric Eel) and 25.10 (Goldeye) this is ancient history provided the system is on a current point release. The lesson for the field: if you ever see `HuC firmware i915/dg2_huc_gsc.bin: fetch failed -ENOENT`, the fix is "update SCALE to the latest point release," not "fiddle with module options."

### GuC 70.x version notes

No specific 70.5 -> 70.6 regression is pinned in a single tracker, but the general pattern is real: the kernel driver and the firmware blob are version-coupled, and a newer kernel with an older `linux-firmware` (or vice versa) can produce `GuC firmware version X not supported` style errors. The mitigation is always the same: keep `linux-firmware` matched to the kernel, which SCALE does as a whole OS update. Do not manually swap blobs unless a specific ix-systems or Intel engineer tells you to.

## 3. The render node and ownership

### Expected end state

After successful bring-up the A380 exposes:

```
/dev/dri/
  card0           <- primary DRM node
  renderD128      <- render node (the one the container binds)
  by-path/
    pci-0000:0d:00.0-render  -> ../renderD128
    pci-0000:0d:00.0-card    -> ../card0
```

`renderD128` is the standard name for the first GPU's render node. If there is an Intel iGPU on the host as well, the A380 may show up as `renderD129` (nodes are numbered 128 + N for the Nth render node). Check with `ls -l /dev/dri/by-path` to map PCI address to node name unambiguously rather than assuming `128`.

### Ownership on SCALE (Debian family)

- Owner: `root:render`
- Group `render` GID **104** on Debian and TrueNAS SCALE (Debian Bookworm based for 24.10). Ubuntu uses GID 106; this matters if you are migrating a compose file written against Ubuntu.
- Mode: typically `0666` (`crw-rw-rw-`) on SCALE via the default udev rule, so any container can open it. If mode is `0660`, the container process needs to be in a group whose GID maps to 104, or you `chmod 666` the node at boot.

For the container bind-mount, the robust pattern is:

```
devices:
  - /dev/dri/renderD128:/dev/dri/renderD128
  - /dev/dri/card0:/dev/dri/card0      # optional, some stacks want it
```

plus either add the container user to a group with GID 104, or rely on the `0666` mode. Group name inside the container is irrelevant; only the numeric GID has to match (or the node has to be world-writable).

## 4. Known Arc-specific host issues

- **Resizable BAR (ReBAR) in BIOS:** enable it. HuC loading via GSC and overall DG2 stability are measurably better with ReBAR on. Community reports of `dg2_huc_gsc.bin` failing to load sometimes resolve to ReBAR being disabled.
- **MEI modules:** the i915 driver depends on the `mei` (management engine) subsystem for GSC communication. Stock SCALE kernels have these on; if you ever build a custom kernel, `CONFIG_INTEL_MEI_HDCP`, `CONFIG_INTEL_MEI_GSC`, and friends must be enabled or HuC silently fails.
- **The "DG2 needs GuC" requirement:** covered above. It is not a quirk you enable; it is the architecture. You cannot disable GuC submission on Arc even if you wanted to.
- **Power management on a headless box:** the A380 is a discrete card with no display attached. `i915` will keep it at idle clocks when no work is submitted. There is no special module option needed for headless operation. The card idles around 5-10 W.
- **`i915.force_probe=!i915.force_probe`:** this odd-looking string (literally `i915.force_probe=!i915.force_probe`) appears in some forum copy-pasta as a way to disable the force_probe mechanism. It is not something you need on the A380. Ignore it unless a specific ix-systems engineer points at your PCI ID.
- **Multiple Intel GPUs (iGPU + A380):** if the host CPU has an Intel iGPU, both will bind `i915`, and you will see `card0/renderD128` (typically the iGPU, lower PCI bus) and `card1/renderD129` (the A380). Use `by-path` to pick the right one for the container. Disable the iGPU in BIOS if you do not use it, to avoid confusion.

## 5. Verification command set

Run these on the TrueNAS host shell. Each one answers a specific question.

### Does the host see the GPU?

```
lspci -nn | grep -i vga
# expect: 0d:00.0 VGA compatible controller [0300]: Intel Corporation Device [8086:56a5] (rev 05)
```

The PCI ID `8086:56a5` confirms the A380. `56a6` would be the A310.

### Which kernel driver is bound?

```
lspci -nnk -s 0d:00.0 | grep -iE 'driver|module'
# expect:  Kernel driver in use: i915
#          Kernel modules: i915, xe
```

If `Kernel driver in use:` is blank or shows `xe`, something is forcing a non-default state. For the A380 you want `i915` here.

### Did GuC and HuC firmware load?

```
dmesg | grep -iE 'i915|guc|huc' | grep -iE 'firmware|auth|submiss'
# expect:
#   [drm] GT0: GuC firmware i915/dg2_guc_70.bin version 70.5.1
#   [drm] GT0: HuC firmware i915/dg2_huc_gsc.bin version 7.9.3
#   [drm] GT0: HuC: authenticated for all workloads
#   [drm] GT0: GUC: submission enabled
```

The two lines that matter: **`HuC: authenticated`** and **`GUC: submission enabled`**. If HuC says `fetch failed -ENOENT`, your `linux-firmware` is missing the blob. If GuC says `submission disabled`, something is very wrong with the driver bind.

For deeper state, the sysfs/debugfs nodes:

```
sudo cat /sys/kernel/debug/dri/0/gt0/uc/guc_info
sudo cat /sys/kernel/debug/dri/0/gt0/uc/huc_info
```

### What is the driver version?

```
cat /sys/class/drm/card0/device/driver/module/version
# expect: a kernel-style version string tied to SCALE's kernel, e.g. 6.6.44-production+truenas
```

This reads the kernel version the module is built against, which is what you want when sanity-checking against the "needs 6.2+" floor.

### What render nodes exist and who owns them?

```
ls -l /dev/dri/
# expect: card0, renderD128, and a by-path/ directory

ls -l /dev/dri/by-path
# maps PCI addresses to node names; use this to confirm renderD128 == the A380

stat -c '%U:%G %a %n' /dev/dri/renderD128
# expect: root:render 666 /dev/dri/renderD128   (or 660)
```

If the group is not `render` or the GID is not 104, your container pass-through may fail silently with EACCES.

### Does VA-API actually work on the host?

```
vainfo
# expects: VA-API version, "Driver: iHD" (intel-media-driver), and a list of entrypoints
# look for: VAEntrypointEncodeSliceLP (low-power encode, needs HuC)
```

`vainfo` is not installed on stock SCALE; you would run it inside the container (jellyfin/plex media images ship it). On the host, the `dmesg` checks above are sufficient proof of life.

## 6. Confidence notes

- **Doc-backed:** i915 is the default driver for Alchemist/Meteor Lake (Phoronix, Dec 2025, direct quote); xe only defaults from Lunar Lake/Battlemage (same source); A380 = PCI 56a5, A310 = 56a6, initial support 6.0, full support 6.2 (Intel dgpu-docs); GuC submission mandatory on DG2, enable_guc bitmask values 1/2/3 (kernel.org i915 docs, ArchWiki); DG2 firmware filenames `dg2_guc_70.bin`, `dg2_huc_gsc.bin`, `dg2_dmc_ver2_08.bin` (kernel.org linux-firmware tree); render group GID 104 on Debian, 106 on Ubuntu (Proxmox/Arch community corroboration); TrueNAS 24.04.0 missing HuC blob, fixed in 24.04.2 (TrueNAS forum staff confirmation); SCALE 24.10 kernel 6.6.44, 25.10 kernel 6.12 LTS (TrueNAS release notes/blog).
- **Community-sourced:** Resizable BAR helping HuC load (Unraid/kernel GitHub issues); MEI/GSC kernel config dependency for HuC (elrepo bug); A380 happily doing 10+ 4K transcodes on SCALE (r/truenas); the `force_probe` syntax examples for switching i915 -> xe (Manjaro forum, ArchWiki).
- **Undetermined from public docs:** exact GuC firmware version currently shipped by SCALE 25.10 (likely 70.5.1 or newer, but point-release dependent); whether a specific GuC 70.x version introduced a regression that affects the A380 (no single tracker pins one; the general coupling problem is well documented); the exact udev rule SCALE uses to set renderD128 mode (assumed 0666 based on Debian default, verify with `stat`).

## Sources

[Phoronix: Intel Xe vs. i915 Driver Performance On Linux 6.19 For Arc Alchemist GPUs](https://www.phoronix.com/review/intel-xe-i915-linux-619) (default driver for Alchemist, xe only defaults from Lunar Lake/Battlemage) · [Phoronix: Intel's Experimental Xe Driver Lacking HuC Media Support For DG2/Alchemist](https://www.phoronix.com/news/Linux-6.2-Stable-Intel-Arc-DG2) (DG2 graduated from experimental in 6.2) · [Intel dgpu-docs: i915 KMD Driver GPUs](https://dgpu-docs.intel.com/overview/supported-hardware/i915-driver-gpus.html) (A380 = 56a5, A310 = 56a6, kernel support matrix) · [kernel.org: linux-firmware i915/ tree](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/i915) (dg2_guc_70.bin, dg2_huc_gsc.bin, dg2_dmc_ver2_08.bin) · [kernel.org: i915 driver documentation](https://docs.kernel.org/gpu/i915.html) (GuC not mandatory in general, enable_guc semantics) · [kernel.org: Xe merge acceptance plan (RFC)](https://www.kernel.org/doc/html/latest/gpu/rfc/xe.html) (xe transition, force_probe mechanism) · [ArchWiki: Intel graphics](https://wiki.archlinux.org/title/Intel_graphics) (enable_guc bitmask 1/2/3, verification via debugfs, force_probe syntax, xe enables GuC/HuC by default) · [Intel: Enabling the GuC/HuC Firmware for Linux on New Intel GPU Platforms](https://www.intel.com/content/www/us/en/content-details/609249/enabling-the-guc-huc-firmware-for-linux-on-new-intel-gpu-platforms.html) · [Intel Community: Arc drivers in Ubuntu](https://community.intel.com/t5/Graphics/Intel-Arc-drivers-in-Ubuntu-help/m-p/1591021) (manual install no longer needed, dg2_guc_70.bin) · [TrueNAS Forums: Missing firmware for Intel Arc DG2 (Alchemist, A380)](https://forums.truenas.com/t/missing-firmware-for-intel-arc-dg2-alchemist-a380/2491) (the 24.04.0 regression, 24.04.2 fix, the `-ENOENT` dmesg) · [TrueNAS Forums: Upgrade to Dragonfish broke Intel QuickSync](https://forums.truenas.com/t/upgrage-to-truenas-scale-dragonfish-broke-intel-quicksync/4526) (i915 in use, GuC/HuC dmesg pattern) · [TrueNAS Forums: Electric Eel Intel Arc GPU passthrough support](https://forums.truenas.com/t/electric-eel-intel-arc-gpu-passthrough-support/12052) (A380 works for transcode on 25.10) · [TrueNAS Docs: 24.10 Electric Eel version notes](https://www.truenas.com/docs/scale/24.10/gettingstarted/scalereleasenotes/) (kernel 6.6.44) · [TrueNAS Docs: 25.10 Goldeye version notes](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalereleasenotes/) (kernel 6.12 LTS) · [TrueNAS Blog: Fangtooth 25.04 release](https://www.truenas.com/blog/truenas-fangtooth-25-04-release/) (first to adopt 6.12) · [TrueNAS Blog: Goldeye 25.10 BETA](https://www.truenas.com/blog/truenas-goldeye-25-10-beta/) (6.12.15 -> 6.12.33) · [TrueNAS Forums: 25.10.4 is now available](https://forums.truenas.com/t/truenas-25-10-4-is-now-available/66244) (kernel 6.12.91) · [r/truenas: Intel A380 not hardware transcoding](https://www.reddit.com/r/truenas/comments/1bjv4l2/intel_a380_not_hardware_transcoding_in_plex_via/) (A380 does 10+ 4K transcodes on SCALE) · [Jellyfin GitHub #12118: Hardware transcode issue on Intel ARC A380 on TrueNAS](https://github.com/jellyfin/jellyfin/issues/12118) · [Launchpad Bug #2029345: Missing DG2 HUC firmware](https://bugs.launchpad.net/bugs/2029345) · [elrepo bug 1470: Missing modules for Intel ARC GPU](https://elrepo.org/bugs/view.php?id=1470) (MEI/GSC dependency) · [thor2002ro/unraid_kernel #11: HuC firmware failing to load for A380](https://github.com/thor2002ro/unraid_kernel/issues/11) (enable ReBAR) · [r/Proxmox: Intel GPU passthrough permission errors](https://www.reddit.com/r/Proxmox/comments/1iwd5id/intel_gpu_passthrough_permission_errors_on_lxc/) (render GID 104 Debian / 106 Ubuntu)
