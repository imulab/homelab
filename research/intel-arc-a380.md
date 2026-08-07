# Intel Arc A380 on TrueNAS SCALE 25.10 for Jellyfin transcoding — verification

> Ad-hoc verification probe (not yet a formal wayfinder ticket). Compiled
> 2026-08-07 via `/research`. Decides whether an Intel Arc A380 is a safe
> purchase to replace the NVIDIA Quadro P400 (Pascal) that #15 proved is broken
> on TrueNAS 25.10's open NVIDIA modules.
>
> Scope: pure research from external docs — no host access, no config edits, no
> decisions. Cross-references the #10 findings (`jellyfin-gpu-passthrough.md`)
> and #15 (`truenas-gpu-mechanism.md`). Verifies three claims made by a separate
> assistant (Gemini); the load-bearing one is claim 3 ("Intel drivers are baked
> in, `/dev/dri` mapping is very clean").

## TL;DR

**Verdict: GO-WITH-CAVEATS.** The A380 is the right card for this pivot and the
"drivers baked in / clean device mapping" claim is **confirmed at the
driver-and-Docker layer**. The caveats are physical (power connector + slot
width are variant-dependent) and an empirical permissions check after install
that costs nothing to run. None of them rise to NO-GO.

The three claims, scored:

- **Claim 1 (GPU isolation = VM PCIe passthrough, not containers): CONFIRMED
  (high confidence).** Already established by prior research; reaffirmed below
  for completeness. Isolating the A380 would **break** container use — do not.
- **Claim 2 (P400 too old for 25.10): CONFIRMED (high confidence).** Already
  established by #15. Reaffirmed in one line.
- **Claim 3 (Intel drivers baked in; `/dev/dri` mapping clean): CONFIRMED at
  the decisive layer (high confidence).** This is the claim justifying the
  purchase, and it holds. Five independent confirmations:
  1. The in-tree Linux **`i915`** kernel driver covers Arc DG2/Alchemist
     (including the A380) — stable since **Linux 6.2**. Intel's own
     `dgpu-docs.intel.com` lists the A380 explicitly. TrueNAS 25.10 ships
     **kernel 6.12 LTS** (6.12.33 at BETA, 6.12.91–6.12.95 in point releases) —
     **far above** the 6.2 floor. No `xe`, no manual install.
  2. This is the **exact opposite** of the NVIDIA situation. NVIDIA's problem in
     #15 is that 25.10's *open* modules reject Pascal. Intel's driver is *open,
     in-tree, and already in the kernel* — there is no open/proprietary fork, no
     toolkit to register, no runtime to install. The whole failure layer that
     broke the P400 does not exist for Intel.
  3. The Jellyfin Intel HWA docs publish the exact compose path: a **`devices:`**
     block mapping `/dev/dri/renderD128` plus a **`group_add:`** for the host
     `render` GID. **No `deploy.resources.reservations.devices`. No runtime.
     No toolkit. No `NVIDIA_*` env.** This is genuinely simpler than the NVIDIA
     block in #10, and for a real structural reason (device-node passthrough vs.
     a vendor runtime hook).
  4. The **linuxserver.io `jellyfin` image ships the iHD userspace driver**
     (`intel-media-driver` / oneVPL) and **auto-grants the `abc` user render
     permissions** — so neither the VAAPI libs nor the render-group dance needs
     provisioning on the TrueNAS host. The "clean" claim holds for userspace
     too.
  5. Real-world A380 + TrueNAS 25.10 + Docker reports confirm the card
     initializes and decodes/encodes; the **failure mode that exists is
     permissions/passthrough, not driver load** — a categorically different
     (and much shallower) problem than NVIDIA's.

**Caveats that could bite after unboxing (all solvable, none fatal):**

- **Power connector is variant-dependent.** The A380 is 75W TBP — right at the
  PCIe slot limit. **Reference design = 8-pin**. **Gunnir later revisions =
  no connector**. **ASRock Low Profile = no connector**. The operator must
  confirm which A380 SKU and whether the TrueNAS box has the matching 6/8-pin
  cable (or buy the no-connector variant).
- **Physical fit is variant-dependent.** The P400 is **single-slot,
  low-profile**. Most A380s are **dual-slot**. A true single-slot low-profile
  A380 exists but is rare; the widely available ASRock Low Profile A380 is
  low-profile but still **dual-slot (39mm thick)**. Confirm the chassis has a
  second slot opening.
- **GuC/HuC firmware must load.** DG2 requires GuC firmware (in
  `/lib/firmware/i915/dg2_guc_*.bin`) for media functions. On kernel 6.12 this
  is in the `linux-firmware` package and loads **automatically** (no
  `enable_guc=2/3` modprobe dance needed on modern kernels) — but it is the one
  thing to grep `dmesg` for if transcodes fail.
- **Resizable BAR should be enabled in the host BIOS.** Arc cards perform
  poorly (and can misbehave) without ReBAR. This is a one-time BIOS toggle.
- **Permissions, not drivers, are the empirical unknown.** A recurring A380-on-
  25.10 failure mode is the app/container lacking write access to the render
  node. After install, run the `vainfo` check (point 4) — if it lists profiles,
  drivers are fine and any remaining issue is the render-group GID.

---

<details>
<summary><b>Claim 1 — GPU Isolation = VM PCIe passthrough, not containers (full detail)</b></summary>

**CONFIRMED (high confidence).** Prior research established this; reaffirmed
because the operator asked, and because it has a direct consequence: **do not
isolate the A380**, or it stops being usable by containers.

TrueNAS docs ("Managing GPUs") describe exactly two functions under
**System Settings → Advanced → GPU Isolation**:

1. **GPU passthrough** — presenting an internal PCI GPU *directly to a VM*.
2. **GPU isolation** — binding the GPU to `vfio-pci` so the *host kernel*
   releases it, preparatory to that VM passthrough.

Both functions are about **VFIO/PCIe passthrough into virtual machines**. They
have nothing to do with Docker containers. Multiple forum threads
("Fangtooth Nvidia GPU passthrough for Apps", "HELP - GPU Passthrough to VM
Issues", "Hardware passthrough of GPU to a VM") consistently state that for
**container Apps** the path is the Docker device/runtime layer (NVIDIA toolkit,
or `/dev/dri` for Intel/AMD), *not* the isolation screen.

**Consequence for the A380:** isolating the A380 would bind it to `vfio-pci`
and **remove** it from `/dev/dri`, breaking both the curated Jellyfin app's
"GPU" dropdown and any Dockhand `/dev/dri/renderD128` mapping. Leave the A380
un-isolated so the i915 driver binds it normally and exposes
`/dev/dri/card0` + `/dev/dri/renderD128` for container device-node passthrough.

Sources: [TrueNAS — Managing GPUs](https://www.truenas.com/docs/scale/22.12/scaletutorials/systemsettings/advanced/managegpuscale/),
[TrueNAS forum — Fangtooth NVIDIA GPU passthrough for Apps](https://forums.truenas.com/t/fangtooth-nvidia-gpu-passthrough-for-apps/40731),
[TrueNAS forum — HELP GPU Passthrough to VM Issues](https://forums.truenas.com/t/help-gpu-passthrough-to-vm-issues/49070).

</details>

<details>
<summary><b>Claim 2 — P400 too old for TrueNAS 25.10 (full detail)</b></summary>

**CONFIRMED (high confidence) — already settled by #15.** Restated in one
line: 25.10's version notes say *"TrueNAS 25.10 switches to open GPU kernel
drivers supporting Turing and newer"* and list *"Pascal, Maxwell, and Volta
architectures are no longer supported"* as a **breaking change**. The Quadro
P400 is Pascal (GP107), no GPU System Processor, so the open `nvidia.ko`
refuses to load. This is the proximate reason for the Intel pivot. See
`truenas-gpu-mechanism.md` ("Pascal reversal" section) for the full evidence.

</details>

<details>
<summary><b>Point 1 — Is i915 / xe actually in the TrueNAS 25.10 kernel? (full detail)</b></summary>

**CONFIRMED: the in-tree `i915` driver covers the A380; no `xe`, no manual
install. High confidence.** This is the keystone of the whole pivot.

**The kernel version ships well above the A380 floor.** TrueNAS 25.10 ("Goldeye")
version notes' Software Component Versions table lists **Linux kernel 6.12 LTS**
— 6.12.33 at BETA, bumped to 6.12.91 (25.10.4) and 6.12.95 (25.10.5) for
security. The A380 (DG2-G11, Alchemist) needs **kernel ≥ 6.0 for initial
support and 6.2 for full/stable support** per Intel's own
`dgpu-docs.intel.com` — TrueNAS 25.10 clears that floor by a wide margin
(kernel 6.12 is ~10 minor releases past 6.2).

**The A380 is explicitly listed as i915-supported.** Intel's
`dgpu-docs.intel.com/overview/supported-hardware/i915-driver-gpus.html` table
includes "Intel® Arc™ A380 Graphics" under desktop models, with the same
"Kernel 6.0 / Ubuntu 24.04+ (6.2)" requirement as the rest of the desktop Arc
line (A310/A380/A580/A750/A770). The `i915` kernel module name is used for
**both** Intel iGPUs and Arc dGPUs — a common source of confusion, confirmed by
the r/truenas A770 thread: *"The i915 driver name is used for both the Intel
iGPU and dGPU series in the kernel version used by TrueNAS."*

**i915, not xe.** The newer `xe` driver is Intel's eventual successor for
discrete GPUs, but for **DG2/Alchemist** (A380) the `i915` path is the mature,
default, recommended one. Phoronix documented DG2 graduating from "experimental"
to **stable in `i915` at Linux 6.2**. The `xe` driver is the one being adopted
for **Battlemage** (Arc B-series) and beyond; forum threads note Battlemage
does not work on current TrueNAS ("Arc Battlemage … is not supported by the
current Truenas version"), but Alchemist (A380) does. There is **no reason to
force `xe`** for the A380; doing so is an advanced, optional, and
rougher-edged path (an Unraid thread documents a user *wanting* to switch an
A380 from i915 to xe — it is a modprobe-parameter exercise, not a requirement).

**Contrast with the NVIDIA situation (#15) — this is the whole point.** The
NVIDIA failure on TrueNAS 25.10 is a *driver-load* failure: 25.10's *open*
NVIDIA modules reject Pascal. There is no analogous failure mode for Intel
because:

- The Intel driver is **open-source and in-tree** — it ships *with* the kernel,
  not as a separately-installed, separately-versioned, open-vs-proprietary fork.
  There is no "TrueNAS chose the open fork and it doesn't cover my GPU" problem
  because there is only one fork.
- There is **no NVIDIA Container Toolkit equivalent to install**. The NVIDIA
  path (#10) needs `nvidia-container-toolkit` + `nvidia-ctk runtime configure`
  + a registered `nvidia` runtime, none of which is default on the appliance.
  The Intel path needs **none of that** — it is pure device-node passthrough
  (point 3).
- The kernel module binds the hardware automatically at boot (assuming firmware
  loads — see the GuC caveat in point 5). The GPU appears at `/dev/dri/card0`
  and `/dev/dri/renderD128` with no operator action beyond physically seating
  the card.

**So "drivers are baked in" (Gemini's claim 3, first half) is literally true
and is the decisive reason the pivot de-risks the box.** The one residual
kernel-side dependency is **firmware** (GuC/HuC blobs), which ships in the
`linux-firmware` package — see point 5 for why this is a non-issue on 25.10's
kernel but the thing to grep `dmesg` for.

Sources: [Intel dgpu-docs — i915-driver GPUs](https://dgpu-docs.intel.com/overview/supported-hardware/i915-driver-gpus.html),
[TrueNAS 25.10 Goldeye Version Notes](https://www.truenas.com/docs/scale/25.10/gettingstarted/versionnotes/),
[Phoronix — Linux 6.2 Stable Intel Arc DG2](https://www.phoronix.com/news/Linux-6.2-Stable-Intel-Arc-DG2),
[Reddit r/truenas — A770 i915 covers iGPU + dGPU](https://www.reddit.com/r/truenas/comments/1fyt2xg/truenas_scale_jellyfin_intel_arc_a770/),
[Unraid forum — A380 i915 vs xe](https://forums.unraid.net/topic/197878-unraid-724-intel-arc-a380-using-i915-driver-would-like-to-use-xe-driver/),
[Kernel.org — drm/i915 docs](https://docs.kernel.org/gpu/i915.html).

</details>

<details>
<summary><b>Point 2 — Does the A380 specifically work for Jellyfin transcoding? (full detail)</b></summary>

**CONFIRMED: yes — H.264/HEVC encode+decode, AV1 decode, and AV1 encode all
work via QSV/VAAPI on the A380. High confidence, primary-source.** The A380 is
Gen 12.5 DG2 / XeHPG (Alchemist), which is exactly the tier Jellyfin's Intel
docs target for Arc.

**Jellyfin's own codec matrix (from the Intel HWA tutorial) maps cleanly onto
the A380:**

| Codec | A380 (Gen 12.5 DG2) | Jellyfin Intel-doc statement |
|---|---|---|
| H.264 decode + encode | YES | "Decoding & Encoding H.264 8-bit — Any Intel GPU that supports QSV" |
| HEVC decode + encode | YES (10-bit) | "Decoding & Encoding HEVC 10-bit — Gen 9.5 Kaby Lake and newer" |
| AV1 **decode** | YES | "Decoding AV1 8/10-bit — Gen 12 Tiger Lake and newer" |
| **AV1 encode** | **YES** | "Encoding AV1 8/10-bit — Gen 12.5 DG2 / ARC A-series and newer" |

The AV1-encode row is the A380's headline feature and the reason it is the
cheap card to buy: Alchemist was the **first consumer-tier GPU with hardware
AV1 encode**, and unlike NVIDIA there is **no concurrent-session cap** (Jellyfin
docs: *"Unlike NVIDIA NVENC, there is no concurrent encoding sessions limit on
Intel iGPU and ARC dGPU."*). This is a real upgrade over the P400, which has no
AV1 encode at all and no HEVC B-frames (#10's caveat).

**Acceleration API: QSV preferred over VAAPI on Arc.** Jellyfin's Intel doc
states *"QSV — Preferred on mainstream GPUs, for better performance"* over
VAAPI, and specifically calls out that *"ARC A-series GPUs use Gen 12.5 XeHPG
architecture, which continues to improve"* under QSV. Both work; QSV is the
recommended selection in **Dashboard → Playback → Transcoding → Hardware
acceleration: Intel QuickSync (QSV)**, with the VA-API device pointed at
`/dev/dri/renderD128`.

**Userspace driver: iHD (not i965).** Jellyfin's docs distinguish:
- **`iHD` driver** → supports **both QSV and VA-API** interfaces. This is the
  one Arc uses. Successful `vainfo` output reads
  *"Driver version: Intel iHD driver for Intel(R) Gen Graphics — 23.1.2"*.
- **`i965` driver** → VA-API only, for older (pre-Gen 11) hardware. Not the A380.

**Real-world confirmation on the Jellyfin/FFmpeg side:** the Jellyfin GitHub
issue #12118 (A380 on TrueNAS SCALE Dragonfish 24.04) shows FFmpeg logs with
`va_openDriver() returns 0` and a successful `hevc (hevc_qsv) -> av1 (av1_qsv)`
transcode path — i.e. QSV HEVC decode feeding QSV AV1 encode. The issue itself
(a duplicate, closed as not planned) was not a hardware/driver defect.

**No AV1-encode caveat worth flagging for a homelab.** Community AV1-QSV encode
on the A380 is described as a stable, mature pipeline on Linux now (r/ffmpeg
"AV1 QSV on Intel Arc Linux, updated and stable pipeline"). Raw AV1 encode
throughput on the A380 is modest (~7–12 fps at 4K per hands-on benchmarks) —
fine for on-the-fly transcode of a single stream, not for mass re-encoding.

Sources: [Jellyfin — HWA Tutorial On Intel GPU](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/),
[Jellyfin GitHub #12118 — A380 hardware transcode on TrueNAS](https://github.com/jellyfin/jellyfin/issues/12118),
[Reddit r/ffmpeg — AV1 QSV on Arc, stable](https://www.reddit.com/r/ffmpeg/comments/1qge8vd/av1_qsv_on_intel_arc_linux_updated_and_stable/),
[Intel community — A380 AV1 in Handbrake](https://community.intel.com/t5/Graphics/Intel-ARC-A380-in-Handbrake-on-Ubuntu-LTS/m-p/1566189).

</details>

<details>
<summary><b>Point 3 — The Compose / Dockhand path for Intel VAAPI (full detail)</b></summary>

**CONFIRMED: genuinely simpler than the NVIDIA block in #10, for a real
structural reason. High confidence, primary-source (Jellyfin docs).**

**The exact compose block Jellyfin's Intel docs publish:**

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin       # (or lscr.io/linuxserver/jellyfin)
    user: 1000:1000
    group_add:
      - '122'                      # ← host `render` GID; check with: getent group render | cut -d: -f3
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    # ... volumes, ports, networks ...
```

That is the **entire** Intel GPU plumbing. Three things to notice, and each is
a concrete delta vs the NVIDIA block in #10:

1. **`devices:` block, NOT `deploy.resources.reservations.devices`.** Intel uses
   plain device-node passthrough — the compose runtime just bind-mounts the
   host's `/dev/dri/renderD128` character device into the container's
   namespace. There is **no `driver: intel`** because there is no vendor
   runtime hook selecting/aliasing the GPU; the kernel already exposes it as a
   normal device file.
2. **No `runtime:` key. No `NVIDIA_VISIBLE_DEVICES`. No
   `NVIDIA_DRIVER_CAPABILITIES`.** None of these exist for Intel. The NVIDIA
   path (#10) needs the `nvidia` container runtime to intercept container
   creation and inject `/dev/nvidia0`, the userspace libs, and the device
   capability env. Intel needs **none** of that machinery — the device node is
   the contract.
3. **`group_add:` for the render GID is the only "extra".** The render node is
   `crw-rw---- root render`. The container process must be a member of the
   host's `render` group (GID is **host-specific** — commonly 107 on
   Debian/Ubuntu, 122 in some Jellyfin doc examples, 44 on others). The linuxserver
   image **auto-handles** this for its default `abc` user (see below), so with
   that image you may not need `group_add` at all — but it is the one knob to
   reach for if `vainfo` reports a permission error.

**Side-by-side, NVIDIA (#10) vs Intel (this doc):**

```yaml
# NVIDIA (#10) — needs host toolkit + registered runtime
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: 1
          capabilities: [gpu]      # mandatory
# + host: nvidia-container-toolkit, nvidia-ctk runtime configure, daemon.json
# + (on TrueNAS 25.10: also must solve the Pascal/open-module driver-load failure)

# Intel (this doc) — pure device-node passthrough
devices:
  - /dev/dri/renderD128:/dev/dri/renderD128
group_add:
  - '<render GID>'                  # optional with linuxserver image
# + host: nothing. Kernel driver + firmware are already in 25.10.
```

**Why Intel is genuinely simpler (the structural reason):**
- NVIDIA GPU access from a container is **mediated by a vendor runtime** that
  rewrites the container's device list and library mounts at create-time. That
  requires the toolkit installed *and* registered with dockerd (`#10` point 2,
  `#15` point 3). On TrueNAS the appliance model makes that install painful
  (`apt` disabled; non-persistent).
- Intel GPU access is **plain Linux device-node permissions**. The kernel
  driver (`i915`) already created `/dev/dri/renderD128`; the container just
  needs it mapped in and the right group membership. Docker has supported
  `--device` and `group_add` since forever; no vendor runtime exists or is
  needed.

**linuxserver.io image removes the remaining friction.** The
`lscr.io/linuxserver/jellyfin` image (the repo's planned image, per #10):
- **Ships the iHD userspace driver** out of the box. Its version history
  includes *"Specify Intel iHD driver versions to avoid mismatched libva
  errors"* and *"adding Intel drivers"* (2019). So the VAAPI/QSV userspace
  stack (`intel-media-driver` / oneVPL) does **not** need provisioning on the
  TrueNAS host — it is in the image.
- **Auto-grants render permissions** to its default `abc` user: *"We will
  automatically ensure the abc user inside of the container has the proper
  permissions to access this device."* This is why the `group_add:` line may be
  unnecessary with this image.
- Confirms the passthrough flag: *"To leverage hardware acceleration you will
  need to mount /dev/dri video device inside of the container"* →
  `--device=/dev/dri:/dev/dri`.
- Rebases to **Ubuntu** (Noble → "Resolute"), which is the distro family
  Jellyfin's Intel docs target.

**Dockhand pass-through risk: lower than NVIDIA's.** #10 flagged an inference
risk that Dockhand might sanitize unknown compose keys (specifically
`deploy.resources.reservations.devices`). The Intel block uses only
**first-class, universally-supported compose keys** (`devices`, `group_add`,
`user`) — these are core Compose-Spec, not GPU-specific, and there is no
plausible reason a wrapper would strip them. So the Dockhand pass-through
concern that applied to the NVIDIA block is **even less of a concern** here.
(Empirical validation on-box is still the final word, per #10/#12 discipline.)

Sources: [Jellyfin — Intel HWA (compose snippet, iHD driver, render GID)](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/),
[linuxserver.io — docker-jellyfin docs (/dev/dri, auto-permissions, iHD driver)](https://docs.linuxserver.io/images/docker-jellyfin/),
[Docker — device mapping & group_add](https://docs.docker.com/reference/compose-file/),
[Repo cross-ref — #10 NVIDIA compose block](./jellyfin-gpu-passthrough.md),
[Repo cross-ref — #15 Dockhand/key-passthrough risk](./truenas-gpu-mechanism.md).

</details>

<details>
<summary><b>Point 4 — The decisive host check (vainfo) (full detail)</b></summary>

**The Intel equivalent of `nvidia-smi` is `vainfo`.** Run it **inside the
running Jellyfin container** after the card is installed and the container is
deployed:

```bash
# Primary check — lists VAAPI profiles the A380 exposes.
docker exec -it jellyfin /usr/lib/jellyfin-ffmpeg/vainfo
```

**What proves VAAPI is functional:** the command exits 0 and prints:

```
libva info: va_openDriver() returns 0
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 23.1.x (xxxxxxx)
...
VAProfile.MPEG2Simple           : VAEntrypoint.VLD
VAProfile.H264Main               : VAEntrypoint.VLD  (decode)
VAProfile.H264High              : VAEntrypoint.EncSlice  (encode)
VAProfile.HEVCMain              : VAEntrypoint.VLD
VAProfile.HEVCMain10            : VAEntrypoint.EncSlice
VAProfile.Av1Profile0           : VAEntrypoint.VLD      (AV1 decode)
VAProfile.Av1Profile0           : VAEntrypoint.EncSlice (AV1 encode)  ← the headline
...
```

The two things that matter: **`va_openDriver() returns 0`** (driver bound) and
the **`iHD`** driver name (not `i965`, not "no device"). If you see AV1
encode/decode entries, the A380's full codec set is reachable.

**Point vainfo at the specific render node** (useful if the host has both an
iGPU and the A380, so the device isn't `renderD128`):

```bash
docker exec -it jellyfin /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```

**Secondary checks (in increasing strength):**

```bash
# 1. Confirm the render node exists and has the render group (run on the HOST):
ls -l /dev/dri
# Expect: crw-rw----+ 1 root render 226, 128 ... renderD128

# 2. Confirm OpenCL/VAAPI init from jellyfin-ffmpeg (proves full userspace stack):
docker exec -it jellyfin /usr/lib/jellyfin-ffmpeg/ffmpeg -v verbose \
  -init_hw_device vaapi=va:/dev/dri/renderD128 -init_hw_device opencl@va

# 3. Live transcode activity — the real end-to-end proof. Run on the HOST while
#    a client is playing a transcoded stream:
intel_gpu_top          # look for the "Video" / "VEBox" engines going non-zero
```

`intel_gpu_top` (from the `intel-gpu-tools` / `intel-gpu-tools` package) is the
visual analogue of `nvidia-smi -l 1`: it shows per-engine occupancy, so you can
see the Video/Encode engines spin up during a transcode. Note: on the TrueNAS
**appliance host** `intel_gpu_top` may not be installed (it is not part of the
base image); the in-container `vainfo` check is the one that always works
because the linuxserver image ships it.

**If `vainfo` fails, the decision tree:**
- *"libva info: va_openDriver() returns -1"* / *"no devices"* → firmware or
  driver didn't load on the host. Grep `dmesg | grep -iE 'i915|guc|huc|dg2'`
  for GuC/HuC load failures (point 5).
- *"va_openDriver() returns -1"* but `ls /dev/dri` shows the node → permissions.
  Add/fix the `group_add:` render GID (point 3), or check the host render group
  actually contains the container user.
- Profiles list fine but a specific codec errors at transcode time → a
  Jellyfin-side codec-checkbox issue (Dashboard → Playback), not a host issue.

Sources: [Jellyfin — Intel HWA verification commands](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/),
[ArchWiki — Intel graphics (enable_guc, intel_gpu_top)](https://wiki.archlinux.org/title/Intel_graphics).

</details>

<details>
<summary><b>Point 5 — A380 gotchas on TrueNAS 25.10 (full detail)</b></summary>

These are the things that could surprise the operator **after** unboxing. Each
is solvable; together they define the GO-WITH-CAVEATS rather than clean GO.

**A. Power connector — variant-dependent (CONFIRM, before/while buying).**
The A380 is a **75W TBP** card — right at the PCIe-slot power limit (75W). That
makes the auxiliary connector a board-partner choice, **not** a fixed spec:
- **Reference design (TechPowerUp):** 1× **8-pin**. Dual-slot.
- **ASRock Challenger ITX OC:** 1× **8-pin**, recommends 500W PSU, dual-slot.
- **Gunnir (later revisions):** **no external connector** — relies on slot
  power only. Earlier Gunnir revisions used 6-pin.
- **ASRock Low Profile A380:** **no external connector**, low-profile bracket,
  but still **dual-slot** (39mm thick).

**Action:** the operator's TrueNAS box likely has a PSU meant for a storage
appliance (the P400 drew ~30W). Before buying, **check whether the PSU has a
free 6-pin or 8-pin PCIe cable**; if not, **buy the ASRock Low Profile or a
no-connector Gunnir variant** that draws purely from the slot. The P400 itself
had no power connector, so the existing cabling may not include one.

**B. Physical fit — slot width is the gotcha (CONFIRM, before buying).**
The P400 is **single-slot, low-profile** — a very slim card. Most A380s are
**dual-slot**, which is the real form-factor mismatch to watch:
- The widely-recommended **ASRock Low Profile A380** is low-profile (bracket)
  but **dual-slot** (39mm) — needs an adjacent empty slot opening at the back
  of the chassis.
- A true **single-slot + low-profile** A380 exists (a niche/custom variant
  reviewed on YouTube) but is rare and harder to source.

**Action:** confirm the chassis has a **two-slot opening** at the PCIe slot the
A380 will occupy. If the chassis is strictly single-slot (some 1U/2U NAS boxes
and SFF cases are), the A380 will not fit and the operator would need the rare
single-slot variant or a different card (e.g., Arc A310 Eco, which is also
Alchemist and also supported by i915, lower performance but smaller).

**C. GuC / HuC firmware — must load, auto-loads on 6.12 (CHECK dmesg if it
fails).** DG2/Arc requires the **GuC** (graphics microcontroller) firmware for
scheduling/submission and the **HuC** (HEVC microcontroller) firmware for HEVC
media offload. On the `i915` driver these are controlled by the `enable_guc`
module parameter; historically users had to set `options i915 enable_guc=2` (GuC
submission) or `=3` (GuC + HuC) in `/etc/modprobe.d/i915.conf`.
- **On modern kernels (6.2+) and especially 6.12**, the defaults flip to
  GuC-submission-on for DG2 and later platforms, so **no manual modprobe config
  is needed** — ArchWiki notes `enable_guc=3` is *"Default for platforms:
  Alder Lake-P (Mobile) and newer"* and DG2/Alchemist falls in that cohort.
- The firmware blobs (`dg2_guc_*.bin`, `dg2_huc_*.bin`, `dg2_dmc_*.bin`) ship
  in the **`linux-firmware`** package, which Debian/Ubuntu-based TrueNAS
  includes. So they are **present** on 25.10.
- **The one thing to do after install:** if `vainfo` shows "no device" or HEVC
  transcodes fail, grep `dmesg | grep -iE 'i915|guc|huc|dg2'` for firmware
  load errors. The classic symptom (Reddit r/jellyfin, A380 on Linux 6.2) is
  stale `dg2*` firmware files in `/lib/firmware/i915/` — solved by ensuring
  `linux-firmware` is current. On a 25.10 fresh install this should already be
  the case.

**D. Resizable BAR — enable in BIOS (one-time toggle).** Arc cards are designed
  assuming ReBAR is on; with ReBAR off, performance degrades and some operations
  misbehave. **Action:** enter the host BIOS, enable "Resizable BAR" / "Above
  4G Decoding" (the P400 did not need this, so it may currently be off).

**E. Host GPU vs iGPU collision — pick the right render node.** If the TrueNAS
box also has an Intel iGPU (common on the Celeron/N100-class boards people run
TrueNAS on), the host will expose **two** render nodes — typically
`/dev/dri/renderD128` (iGPU) and `/dev/dri/renderD129` (the A380), or vice
versa. The Jellyfin compose `devices:` line must map the **A380's** node, not
the iGPU's. A TrueNAS forum thread (Intel Arc A310 on 25.10) shows exactly this
confusion: the user mapped `renderD129` and playback stopped immediately, then
`renderD128` and it hung — because they had the wrong node. **Action:** after
install, run `ls -l /dev/dri` and `intel_gpu_top` (or `vainfo --device
/dev/dri/renderD12X` for each) to identify which node is the A380, and pin that
in compose.

**F. Permissions — the recurring empirical gotcha.** The single most common
A380-on-TrueNAS-25.10 failure mode in the forums is **not** a driver problem —
it is the app/container lacking write access to the render node. The Plex-on-
25.10.1 thread ("Goldeye despite Intel Arc A310, decode works, encode unclear")
traces exactly this: the iHD driver loaded, decode worked, but encode silently
failed because *"The dataset containing /dev/dri had read-only permissions."*
Fix: grant write permission / correct render-group membership. This is
**categorically shallower** than the NVIDIA Pascal failure (which is a hard
driver-load failure with no in-OS fix short of a community proprietary-driver
installer). It is also exactly what the `vainfo` check in point 4 catches.

**G. Not a gotcha, but worth stating: no toolkit, no runtime, no
host-side package install.** Unlike the NVIDIA path in #10 (which needs
`nvidia-container-toolkit` + `nvidia-ctk runtime configure`, and on 25.10 also
a workaround for the open-module rejection of Pascal), the Intel path needs
**zero** host-side package installation. The kernel driver and firmware ship
with TrueNAS; the userspace driver ships in the linuxserver image. The only
"install" is physically seating the card and (optionally) a BIOS toggle.

Sources: [TechPowerUp — Arc A380 specs](https://www.techpowerup.com/gpu-specs/arc-a380.c3913),
[Tom's Hardware — Gunnir A380 drops 6-pin](https://www.tomshardware.com/news/gunnir-index-a380-ditches-6-pin),
[HotHardware — ASRock Low Profile A380 (no external power, dual-slot)](https://hothardware.com/news/asrocks-first-low-profile-gpu-is-an-intel-arc-a380),
[Tweaktown — ASRock Low Profile A380](https://www.tweaktown.com/news/92226/asrock-launches-low-profile-intel-arc-a380-gpu-that-doesnt-require-external-power/index.html),
[ASRock — Arc A380 Challenger ITX OC](https://www.asrock.com/Graphics-Card/Intel/Intel%2520Arc%2520A380%2520Challenger%2520ITX%25206GB%2520OC/index.gr.asp),
[ArchWiki — Intel graphics (enable_guc, GuC/HuC firmware)](https://wiki.archlinux.org/title/Intel_graphics),
[Reddit r/jellyfin — A380 GuC/HuC/DMC not loading on Linux 6.2](https://www.reddit.com/r/jellyfin/comments/12pvoe/intel_arc_a380_on_linux_62_not_loading_guchucdmc/),
[Level1Techs — Update Intel Arc firmware on Linux](https://forum.level1techs.com/t/remember-to-update-your-intel-arc-firmware-on-linux/208736),
[TrueNAS forum — Arc A310 on 25.10.1 (decode works, encode permissions fix)](https://forums.truenas.com/t/truenas-scale-25-10-1-goldeye-despite-intel-arc-a310-decode-works-encode-unclear-solution/63131),
[TrueNAS forum — Setting up HW transcode on Jellyfin with Intel Arc (renderD128 vs D129)](https://forums.truenas.com/t/setting-up-hardware-transcode-on-jellyfin-intel-arc/65789),
[TrueNAS forum — Intel Arc A380 thread](https://forums.truenas.com/t/intel-arc-a380/21685),
[Reddit r/selfhosted — Jellyfin HW accel Docker A380 (firmware-linux-nonfree fix)](https://www.reddit.com/r/selfhosted/comments/196k4vd/jellyfin_hardware_acceleration_on_docker_intel/).

</details>

## Confidence notes

- **Primary-source, highest confidence:** Point 1 — Intel's `dgpu-docs` lists
  the A380 under i915 with "Kernel 6.0 / 6.2 full support"; TrueNAS 25.10
  version notes list kernel **6.12 LTS** (6.12.33 → 6.12.95). The A380 floor is
  cleared by ~10 kernel minor versions. Point 3 — Jellyfin's own Intel HWA doc
  publishes the exact `devices:` + `group_add:` compose block; the structural
  difference from NVIDIA (no runtime, no toolkit, device-node passthrough) is
  inherent to how VAAPI works, not a matter of opinion. Point 4 — `vainfo`
  command and expected output quoted from Jellyfin's docs.
- **High confidence, multi-source:** Point 2 — A380 codec coverage (H.264/HEVC/
  AV1 decode + AV1 encode) from Jellyfin's codec matrix + a real GitHub issue
  showing QSV HEVC→AV1 working + community AV1-QSV stability reports. Point 5
  (firmware) — ArchWiki + multiple A380-on-Linux reports converge on "GuC
  firmware required, ships in linux-firmware, auto-loads on modern kernels."
- **High confidence, host-state dependent:** Claims 1 and 2 reaffirmed from
  prior research (#15) and TrueNAS docs.
- **Caveats that are confirmed facts but variant-dependent (flag, do not
  block):** power connector and slot width (5A, 5B) — these depend on which
  exact A380 SKU the operator buys. ReBAR (5D) and the iGPU-collision render-
  node pick (5E) are one-time config checks after install.
- **Lower confidence / community-sourced:** the specific A380-on-25.10 forum
  reports are mixed but **all of them are permissions/passthrough failures,
  not driver failures** — which is itself the load-bearing finding (the driver
  layer works; the failure layer that broke the P400 does not exist for Intel).
- **Cannot confirm from docs alone (→ on-box after purchase):** whether the
  operator's *specific* TrueNAS chassis/PSU has a free PCIe power cable and a
  two-slot opening; whether ReBAR is currently on; whether the host will expose
  the A380 as `renderD128` or `renderD129`. All are cheap to check and do not
  require the card to be in hand for the first two.

## Sources

- [Jellyfin — HWA Tutorial On Intel GPU (QSV/VAAPI, compose block, codec matrix, vainfo)](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/)
- [Jellyfin — Hardware acceleration overview](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)
- [linuxserver.io — docker-jellyfin docs (/dev/dri, iHD driver, auto-permissions)](https://docs.linuxserver.io/images/docker-jellyfin/)
- [Intel dgpu-docs — i915-driver GPUs (A380 listed, kernel 6.0/6.2)](https://dgpu-docs.intel.com/overview/supported-hardware/i915-driver-gpus.html)
- [TrueNAS SCALE 25.10 (Goldeye) Version Notes (kernel 6.12 LTS, Docker Engine 28.3.1, NVIDIA open modules / Pascal dropped)](https://www.truenas.com/docs/scale/25.10/gettingstarted/versionnotes/)
- [TrueNAS — Managing GPUs (GPU isolation = VM PCIe passthrough)](https://www.truenas.com/docs/scale/22.12/scaletutorials/systemsettings/advanced/managegpuscale/)
- [Phoronix — Linux 6.2 Stable Intel Arc DG2 (i915)](https://www.phoronix.com/news/Linux-6.2-Stable-Intel-Arc-DG2)
- [Kernel.org — drm/i915 driver docs](https://docs.kernel.org/gpu/i915.html)
- [ArchWiki — Intel graphics (GuC/HuC firmware, enable_guc, intel_gpu_top)](https://wiki.archlinux.org/title/Intel_graphics)
- [Jellyfin GitHub #12118 — A380 hardware transcode on TrueNAS Dragonfish (QSV HEVC→AV1)](https://github.com/jellyfin/jellyfin/issues/12118)
- [TrueNAS forum — Jellyfin on 25.10.0.1 Goldeye AV1 GPU (Arc = obvious AV1 choice)](https://forums.truenas.com/t/jellyfin-on-truenas-scale-25-10-0-1-goldeye-av1-gpu/59807)
- [TrueNAS forum — Setting up HW transcode on Jellyfin with Intel Arc (renderD128 vs D129)](https://forums.truenas.com/t/setting-up-hardware-transcode-on-jellyfin-intel-arc/65789)
- [TrueNAS forum — Arc A310 on 25.10.1 (decode works, encode = permissions fix, not driver)](https://forums.truenas.com/t/truenas-scale-25-10-1-goldeye-despite-intel-arc-a310-decode-works-encode-unclear-solution/63131)
- [TrueNAS forum — Intel Arc A380 thread](https://forums.truenas.com/t/intel-arc-a380/21685)
- [TrueNAS forum — Fangtooth NVIDIA GPU passthrough for Apps](https://forums.truenas.com/t/fangtooth-nvidia-gpu-passthrough-for-apps/40731)
- [Reddit r/truenas — Jellyfin Intel Arc A770 (i915 covers iGPU + dGPU)](https://www.reddit.com/r/truenas/comments/1fyt2xg/truenas_scale_jellyfin_intel_arc_a770/)
- [Reddit r/selfhosted — Jellyfin HW accel Docker A380 (firmware-linux-nonfree fix)](https://www.reddit.com/r/selfhosted/comments/196k4vd/jellyfin_hardware_acceleration_on_docker_intel/)
- [Reddit r/jellyfin — A380 GuC/HuC/DMC not loading on Linux 6.2](https://www.reddit.com/r/jellyfin/comments/12pvoe/intel_arc_a380_on_linux_62_not_loading_guchucdmc/)
- [Reddit r/ffmpeg — AV1 QSV on Intel Arc Linux, stable pipeline](https://www.reddit.com/r/ffmpeg/comments/1qge8vd/av1_qsv_on_intel_arc_linux_updated_and_stable/)
- [Unraid forum — A380 i915 vs xe driver](https://forums.unraid.net/topic/197878-unraid-724-intel-arc-a380-using-i915-driver-would-like-to-use-xe-driver/)
- [Level1Techs — Update Intel Arc firmware on Linux](https://forum.level1techs.com/t/remember-to-update-your-intel-arc-firmware-on-linux/208736)
- [TechPowerUp — Arc A380 specs](https://www.techpowerup.com/gpu-specs/arc-a380.c3913)
- [Tom's Hardware — Gunnir A380 drops 6-pin](https://www.tomshardware.com/news/gunnir-index-a380-ditches-6-pin)
- [HotHardware — ASRock Low Profile A380 (no external power, dual-slot)](https://hothardware.com/news/asrocks-first-low-profile-gpu-is-an-intel-arc-a380)
- [Tweaktown — ASRock Low Profile A380](https://www.tweaktown.com/news/92226/asrock-launches-low-profile-intel-arc-a380-gpu-that-doesnt-require-external-power/index.html)
- [ASRock — Arc A380 Challenger ITX OC](https://www.asrock.com/Graphics-Card/Intel/Intel%2520Arc%2520A380%2520Challenger%2520ITX%25206GB%2520OC/index.gr.asp)
- [Repo cross-reference — `research/jellyfin-gpu-passthrough.md` (#10 NVIDIA compose block)](./jellyfin-gpu-passthrough.md)
- [Repo cross-reference — `research/truenas-gpu-mechanism.md` (#15 mechanism + Pascal reversal)](./truenas-gpu-mechanism.md)
