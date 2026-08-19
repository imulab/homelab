# Intel Arc A380 BIOS requirements briefing

> Resolves wayfinder ticket #01. Compiled 2026-08-11 via `/research` probe.
> Canonical identity: **Intel Arc A380** (ACM-G11 die, Alchemist / DG2 family, 6 GB GDDR6, 96-bit bus). Low-profile, ~75 W, popular as a Quick Sync transcode / display headless card in SFF and homelab builds. Specs: [TechPowerUp arc-a380.c3913](https://www.techpowerup.com/gpu-specs/arc-a380.c3913).

## TL;DR (answer to "do I need to enable Above 4G Decoding?")

**Yes. Enable it.** Above 4G Decoding is required for the A380 to map its 64-bit prefetchable PCI memory regions (the VRAM aperture and the rest of the card's MMIO) above the legacy 32-bit, sub-4 GB window. With it off the kernel cannot place the BARs and the driver either fails to init or the card limps along through a 256 MB window. Intel's own enablement guide lists Above 4G Decoding as a required step alongside Re-Size BAR Support and UEFI boot. [intel.com 000090831](https://www.intel.com/content/www/us/en/support/articles/000090831/graphics.html)

While you are in the BIOS, also enable **Re-Size BAR Support** and confirm **CSM/Legacy is off (pure UEFI)**. Recent firmware does NOT default these on for ASRock/ASUS/MSI consumer and SFF boards; you must set them explicitly. Full rationale, failure modes, and verification commands below.

## Per-setting directives

| Setting | Directive | One-line justification |
|---|---|---|
| **Above 4G Decoding** | **Required** | The A380's 64-bit prefetchable BARs (including the VRAM aperture) do not fit in the legacy 32-bit MMIO window; without Above 4G the BIOS cannot assign them and the card either fails to enumerate cleanly or runs through a 256 MB sliver. Intel lists it as a mandatory enable step. |
| **Re-Size BAR Support (ReBAR)** | **Strongly recommended, treat as required** | Intel's official line is "must be enabled for optimal performance," but real-world reports range from severe slowdown to outright driver failure. For a transcode/compute box doing real VRAM work, you do not want the 256 MB fallback aperture. |
| **CSM / Legacy boot** | **Required: OFF (pure UEFI)** | ReBAR and Above 4G are gated behind UEFI mode on essentially all boards; CSM/Legacy must be disabled or the toggles are greyed out or silently ineffective. Intel states this explicitly. |
| **Primary Display (PEG/PCI vs iGPU)** | **Set explicitly if the board has an iGPU** | On platforms with onboard graphics (most Photonicat-class SFF boxes are dGPU-only, but many mini-PC host boards have an iGPU), set Primary Display to the PCIe slot (PEG) so the A380 is the boot VGA. Leave iGPU enabled only if you need it for coexistence. |
| **iGPU Multi-Monitor** | **Conditional** | Only relevant if you keep the onboard iGPU active alongside the A380 (e.g. driving a local panel from the iGPU while the A380 does transcode). Enable it to keep both adapters alive; otherwise disable to avoid VGA arbitration overhead. |
| **SR-IOV** | **Not needed** | Only relevant for virtualization with multiple passthrough guests. A single-container passthrough does not use it; leave at default. |
| **Secure Boot / VT-d / IOMMU** | **Set per your passthrough plan, not A380-specific** | Enable VT-d/IOMMU if you intend to pass the card through to a container or VM. Not a BIOS-correctness requirement for the A380 itself. |

## Why Above 4G is required (the mechanism)

PCI BARs (Base Address Registers) tell the OS where a card's memory-mapped regions live. Legacy x86 BIOS hands out 32-bit MMIO addresses below 4 GB, and that pool is small (often a few hundred MB, shared across all PCIe devices). The A380, like all DG2/Alchemist parts, advertises its VRAM aperture as a **64-bit prefetchable BAR** (BAR2) and wants it resized up toward the full local-memory size (the kernel commonly targets 8 GB on these parts: `Failed to resize BAR2 to 8192M`). A 64-bit prefetchable BAR can only be placed above 4 GB, which requires the firmware's **Above 4G Decoding** to be on. With it off:

- The BIOS either fails to assign the BAR entirely (`BAR 2: failed to assign [mem size 0x20000000 64bit pref]`), or
- It carves out a tiny 256 MB fallback window, and every VRAM access from the CPU has to page through that aperture, which is what produces the catastrophic performance hit on Arc specifically.

This is the same reason every modern dGPU (ARC, and SAM-era Radeon/GeForce) wants Above 4G. It is not Arc-unique in principle, but Arc is far less tolerant of the 256 MB fallback than NVIDIA/AMD cards are. [Red Hat Bugzilla 2192989](https://bugzilla.redhat.com/show_bug.cgi?id=2192989)

## ReBAR: required, performance-only, or irrelevant?

Intel's own documentation is the source of the "only performance" framing. The support article says Resizable BAR "must be enabled for **optimal performance** in all applications using Intel Arc A-series Graphics" [intel.com 000090831](https://www.intel.com/content/www/us/en/support/articles/000090831/graphics.html). At Arc's launch Intel publicly told buyers on non-ReBAR platforms to "stick with other vendors" [TechPowerUp](https://www.techpowerup.com/298599/resizable-bar-a-must-for-arc-alchemist-stick-with-other-vendors-if-you-lack-it-intel), and the same article quantifies the penalty as "as high as 40 percent."

The kernel-side reality, however, is harsher than "just slower":

- The Arch Wiki reports that with ReBAR off, users hit **bus errors** launching `vainfo`, `mpv`, `falkon` and other VAAPI/Media consumers (`1234 bus error : vaapi`). For a transcode box that is "the feature does not work," not "the feature is slow." [Arch Wiki Intel_graphics](https://wiki.archlinux.org/title/Intel_graphics)
- The IGCIT community tracker has reports of the Arc driver being **inoperative** without ReBAR (system freezes on driver install). [IGCIT #315](https://github.com/IGCIT/Intel-GPU-Community-Issue-Tracker-IGCIT/issues/315)
- On Linux the kernel refuses to resize at runtime when Above 4G/ReBAR are BIOS-off, logging `i915 ... [drm] Failed to resize BAR2 to 8192M (-ENOSPC)` and `BAR 2: failed to assign`. [Red Hat Bugzilla 2192989](https://bugzilla.redhat.com/show_bug.cgi?id=2192989)

Authoritative position to take: **on paper performance-only; in practice, treat as required** for any workload that actually touches VRAM (transcode, compute, ML). For a pure "display a console" role it may limp by, but you bought an A380 to transcode, so turn it on. The kernel patch that added DG2 ReBAR says it plainly: "Starting from DG2 we will have resizable BAR support for device local-memory" [Phoronix via WCCFTech](https://wccftech.com/intel-arc-alchemist-gpus-to-get-resizable-bar-support-on-linux-platforms/), i.e. the local-memory access model on DG2 is built around ReBAR.

## Interaction: Above 4G is a prerequisite for ReBAR

On ASRock, ASUS and MSI boards (and effectively all UEFI firmware), **ReBAR cannot be enabled unless Above 4G Decoding is on first**, and both require **pure UEFI mode (CSM/Legacy disabled)**. If the ReBAR toggle is missing or greyed out, the cause is almost always: CSM still on, or Above 4G still off, or the firmware too old to expose ReBAR at all. The ASRock FAQ documents the exact path: BIOS, Advanced, Chipset Configuration, Above 4G Decoding, Enabled. [ASRock FAQ 498](https://www.asrock.com/support/faq.asp?id=498)

## Recent BIOS defaults (post-firmware-update reality)

You just flashed a newer BIOS. **Do not assume the new defaults help you.** Across the major consumer/SFF vendors:

- **Above 4G Decoding** is **off by default** on the vast majority of ASRock, ASUS and MSI boards, including recent 2024/2025 firmware. Some newer ASUS builds tie it to an "Auto" state and grey it out behind ReBAR, but you should not rely on auto-configuration. [r/pcmasterrace](https://www.reddit.com/r/pcmasterrace/comments/1b4sy75/what_is_above_4g_decoding_resize_bar_support/)
- **Re-Size BAR Support** defaults to **Auto/Disabled** and must be explicitly set to Enabled or Auto-with-ReBAR.
- **CSM** defaults vary; many recent boards ship UEFI-only, but SFF/industrial boards (the kind Photonicat-class host boards are based on) sometimes still default CSM on for legacy payload compatibility. Verify.

Known SFF/mini quirks to watch for on these platforms:

- **ReBAR toggle hidden/greyed out** until you switch from Legacy/CSM to UEFI-only. This is the single most common "I updated the BIOS and still can't find ReBAR" cause. [r/ASRock](https://www.reddit.com/r/ASRock/comments/1h4amdn/not_seeing_resizable_bar_in_bios/)
- **Older boards (Z370/Z390/Z490 era and earlier)** may not expose ReBAR in the GUI at all without a BIOS update, and a few older chipsets need a UEFI module patch (the community `ReBarUEFI` tool) to unlock it. If your host is one of these, no amount of menu hunting will find it without the patch. [github.com/xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI)
- **SFF/industrial boards** sometimes omit the Above 4G toggle entirely from the visible menu; it may exist only as a hidden NVRAM variable. If the menu genuinely lacks it, that is a board-firmware limitation, not a setting you missed.

## Failure-mode symptom list (BIOS-problem vs driver-problem triage)

Use this during bring-up to tell "BIOS config wrong" apart from "driver/firmware wrong."

### If Above 4G Decoding is off (or CSM is on, blocking Above 4G)

- **lspci**: the A380's 64-bit prefetchable BAR shows as `[disabled]` or with a tiny size, e.g. `Region 2: Memory at ... (64-bit, prefetchable) [disabled] [size=256M]`.
- **dmesg**: `pci 0000:NN:00.0: BAR 2: no space for [mem size 0x20000000 64bit pref]` followed by `BAR 2: failed to assign [mem size 0x20000000 64bit pref]`. [Red Hat Bugzilla 2192989](https://bugzilla.redhat.com/show_bug.cgi?id=2192989)
- **i915 init**: the DRM layer logs `[drm] Failed to resize BAR2 to 8192M (-ENOSPC)` and either binds the card in a degraded state or refuses to bring up outputs.
- **POST**: on some boards the system fails to POST or hangs at the VGA LED. More commonly it POSTs but the card is mis-enumerated.
- **Windows-side equivalent**: card shows as **Microsoft Basic Display Adapter**, or Device Manager shows **Code 31** ("driver trying to start is not the same as the driver for the POSTed display adapter") or **Code 43**.

### If ReBAR is off specifically (Above 4G on, ReBAR off)

- Card enumerates and the driver loads, **but**:
  - `vainfo`, `mpv`, any VAAPI/Media SDK consumer exits with **bus error** (`1234 bus error : vaapi`). [Arch Wiki](https://wiki.archlinux.org/title/Intel_graphics)
  - Transcode throughput collapses; CPU access to VRAM is bottlenecked through the 256 MB aperture.
  - In worst cases (community reports) the driver is "inoperative" and the machine freezes after driver install. [IGCIT #315](https://github.com/IGCIT/Intel-GPU-Community-Issue-Tracker-IGCIT/issues/315)
  - Up to ~40% general graphics performance penalty. [TechPowerUp](https://www.techpowerup.com/298599/resizable-bar-a-must-for-arc-alchemist-stick-with-other-vendors-if-you-lack-it-intel)

### If Primary Display is wrong on a board with an iGPU

- System POSTs but blanks, or outputs go to the iGPU instead of the A380. Fix in BIOS, Primary Display, set to PEG/PCI.

### Quick triage rule

- **Card not seen at all / BAR assignment errors / Basic Display Adapter** -> BIOS problem (Above 4G off or CSM on). Fix BIOS first.
- **Card seen, driver loads, but transcode/bus errors/perf terrible** -> ReBAR off. Fix BIOS.
- **Card seen, ReBAR on, driver loads, but encode fails or firmware missing** -> driver/firmware problem (GuC/HuC firmware, kernel version). That is a different ticket.

## Verify it is on (BIOS and post-boot checks)

### In BIOS

Reboot into the firmware (DEL on most boards). Confirm, in order:

1. Boot Mode is **UEFI** (CSM/Legacy **disabled**).
2. **Above 4G Decoding** is **Enabled** (or Auto, if Enabled is not offered).
3. **Re-Size BAR Support** is **Enabled** (or Auto).
4. Save and exit (F10). Changing CSM often forces a re-check of the other two, so verify all three in the same pass.

### From Linux, once the card is in and the system is up

Find the card:

```
lspci -nn | grep -i vga
lspci -d ::0300 -v
```

Take the bus address (e.g. `01:00.0`) and inspect the BAR. With ReBAR on you should see a large prefetchable Region 2 (multi-GB), not 256 MB:

```
lspci -v -s 01:00.0 | grep -i -e prefetchable -e "Region 0" -e "Region 2"
```

Confirm the BAR is actually resizable and what size the kernel assigned:

```
cat /sys/bus/pci/devices/0000:01:00.0/resource | head
# column 1 = start, column 2 = end; (end - start + 1) is the BAR size
```

The `resize` file lists the sizes the card can expose. With ReBAR working you should see 256M, 512M, 1G, 2G, 4G, 8G supported, and the current BAR2 sized at multi-GB:

```
cat /sys/bus/pci/devices/0000:01:00.0/resize
```

Check the kernel log for the i915 bind and any BAR failures:

```
dmesg | grep -i -e i915 -e 'BAR' -e 'drm'
```

A healthy bring-up has no `failed to assign` or `Failed to resize BAR2` lines. If you see them, the BIOS setting did not take (re-verify CSM/Above 4G/ReBAR).

Optional: confirm GuC/HuC firmware loaded (needed for full Quick Sync/media functionality):

```
dmesg | grep -i -e guc -e huc
```

## Confidence notes

- **Doc-backed (Intel + vendor):** Above 4G Decoding and Re-Size BAR Support are both listed as required enable steps in Intel's Resizable BAR support article; CSM must be disabled and UEFI enabled; ASRock FAQ documents the BIOS path. [intel.com 000090831](https://www.intel.com/content/www/us/en/support/articles/000090831/graphics.html), [ASRock FAQ 498](https://www.asrock.com/support/faq.asp?id=498)
- **Doc-backed (kernel/distro):** Arch Wiki documents the ReBAR-off bus-error failure mode for VAAPI consumers (`vainfo`, `mpv`, `falkon`). Red Hat Bugzilla 2192989 captures the exact `BAR 2: failed to assign` and `Failed to resize BAR2 to 8192M (-ENOSPC)` dmesg strings. Phoronix/WCCFTech report the kernel patch text stating DG2 has resizable BAR support for device local-memory. [Arch Wiki](https://wiki.archlinux.org/title/Intel_graphics), [Red Hat Bugzilla 2192989](https://bugzilla.redhat.com/show_bug.cgi?id=2192989), [WCCFTech](https://wccftech.com/intel-arc-alchemist-gpus-to-get-resizable-bar-support-on-linux-platforms/)
- **Community-sourced:** Intel's public "stick with other vendors" guidance and the ~40% performance penalty figure (TechPowerUp); driver-inoperative-without-ReBAR reports (IGCIT #315); Above 4G defaulting to off on ASRock/ASUS/MSI firmware (r/pcmasterrace, r/ASRock); ReBAR toggle being greyed out until CSM is disabled (multiple ASRock/ASUS forum threads). [TechPowerUp](https://www.techpowerup.com/298599/resizable-bar-a-must-for-arc-alchemist-stick-with-other-vendors-if-you-lack-it-intel), [IGCIT #315](https://github.com/IGCIT/Intel-GPU-Community-Issue-Tracker-IGCIT/issues/315)
- **Undetermined from public docs:** the exact BIOS defaults of the user's specific flashed firmware (must be verified in the BIOS itself); whether the user's SFF host board exposes a visible Above 4G toggle at all (some industrial SFF boards hide it); the precise default BAR2 fallback size on the specific A380 SKU (commonly 256 MB, but card-specific).

## Sources

[intel.com 000090831, What Is Resizable BAR and How Do I Enable It](https://www.intel.com/content/www/us/en/support/articles/000090831/graphics.html) · [TechPowerUp, Resizable BAR a Must for Arc Alchemist](https://www.techpowerup.com/298599/resizable-bar-a-must-for-arc-alchemist-stick-with-other-vendors-if-you-lack-it-intel) · [Arch Wiki, Intel graphics (ReBAR bus-error failure mode)](https://wiki.archlinux.org/title/Intel_graphics) · [Red Hat Bugzilla 2192989, Cannot enable PCIe Resizable BAR on Intel ARC DG2](https://bugzilla.redhat.com/show_bug.cgi?id=2192989) · [WCCFTech, Intel Arc Alchemist Resizable BAR Support on Linux (kernel patch text)](https://wccftech.com/intel-arc-alchemist-gpus-to-get-resizable-bar-support-on-linux-platforms/) · [IGCIT Intel-GPU-Community-Issue-Tracker #315, Arc driver inoperative without Resizable BAR](https://github.com/IGCIT/Intel-GPU-Community-Issue-Tracker-IGCIT/issues/315) · [Intel Community, Arc A380 Resizable BAR / Arc Control issues](https://community.intel.com/t5/Graphics/Intel-arc-a380-Resizable-BAR-Intel-Arc-Control-issues/td-p/1600801) · [ASRock FAQ 498, Enabling Above 4G Decoding / Resizable BAR](https://www.asrock.com/support/faq.asp?id=498) · [ASUS FAQ 1045574, iGPU Multi-Monitor](https://www.asus.com/support/faq/1045574/) · [TechPowerUp GPU DB, Arc A380](https://www.techpowerup.com/gpu-specs/arc-a380.c3913) · [r/ASRock, ReBAR toggle greyed out until CSM disabled](https://www.reddit.com/r/ASRock/comments/1h4amdn/not_seeing_resizable_bar_in_bios/) · [r/pcmasterrace, Above 4G Decoding defaults to off](https://www.reddit.com/r/pcmasterrace/comments/1b4sy75/what_is_above_4g_decoding_resize_bar_support/) · [github.com/xCuri0/ReBarUEFI (UEFI module patch for older boards)](https://github.com/xCuri0/ReBarUEFI) · [Proxmox Forum, Reduced BAR on 8.1 with Intel Arc A380](https://forum.proxmox.com/threads/reduced-bar-on-8-1-with-intel-arc-a380.144193/) · [Phoronix, Intel Preparing Resizable BAR Support for DG2/Alchemist](https://www.phoronix.com/news/Intel-DG2-ReBAR-Linux-Prepare) · [HWcooling, Intel Arc graphics require PCIe Resizable BAR for performance](https://www.hwcooling.net/en/intel-arc-graphics-require-pcie-resizable-bar-for-performance/)
