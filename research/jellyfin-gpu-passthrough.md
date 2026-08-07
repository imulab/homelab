# Jellyfin GPU passthrough via Dockhand/Compose

> Resolves wayfinder ticket #10. Compiled 2026-08-07 via `/research` probe.
> Scope: how to give the Jellyfin container exclusive use of an NVIDIA Quadro
> P400 (Pascal GP107) for NVENC transcoding, when the container is driven by
> Dockhand (a Docker Compose UI) on TrueNAS SCALE 25.10. Research only — no
> host access, no config edits. Host-side prerequisite *inspection* is handed to
> ticket #12.

## TL;DR

- **Compose syntax (point 1):** Use the modern Compose-Spec form
  `deploy.resources.reservations.devices` with `driver: nvidia`. This is what
  Jellyfin's own docs and Docker's own docs publish today; the legacy
  `runtime: nvidia` + `NVIDIA_VISIBLE_DEVICES` env form is deprecated and only
  kept for older Docker/Compose. linuxserver.io's `jellyfin` image auto-sets
  `NVIDIA_DRIVER_CAPABILITIES`, so you do **not** pass env vars for it — you
  only declare the device reservation. With a single GPU, `count: 1` is
  preferred over `device_ids` (and `count`/`device_ids` are mutually
  exclusive).
- **Host dependency (point 2):** The `nvidia-container-toolkit` must be
  installed **and** registered with dockerd via `nvidia-ctk runtime configure
  --runtime=docker`. It is **not** present by default on TrueNAS SCALE 25.10 in
  a form usable by custom/Compose containers — and `apt` is disabled on the
  appliance host. TrueNAS ships the **driver** (570.172.08, open kernel
  modules) via the web UI for its own Apps, but the **container toolkit / nvidia
  runtime** for ad-hoc Docker is a separate manual install that is not
  officially supported on the appliance. **This is the single biggest unknown —
  see "Host availability" and hand verification to #12.**
- **Dockhand compatibility (point 3) — the gitops risk:** Dockhand's manual
  does **not** model `deploy.resources.reservations.devices`, GPU passthrough,
  `runtime`, or any device/runtime selection in its UI. Crucially, it also does
  **not** document any sanitization or stripping of compose keys: it is
  described as a wrapper that reads your `docker-compose.yml` from disk and
  hands it to the `docker compose` CLI, adding only flags like `--build`.
  **Strong inference (not doc-stated):** unknown Compose keys therefore flow
  through to Docker Compose verbatim, so `deploy.resources.reservations.devices`
  should be honored — but this is **inference, not a Dockhand guarantee**, and
  must be validated empirically on the box. No UI/runtime selector exists.
- **NVENC enablement (point 4):** In Jellyfin, **Dashboard → Playback →
  Transcoding → Hardware acceleration: NVIDIA NVENC**, then tick H.264 and
  HEVC. The P400 **does** H.264 + HEVC 10-bit encode and H.264/HEVC decode. The
  P400-specific caveat (per NVIDIA's own matrix): **HEVC B-frame support is NO
  on the P400** — quality/bitrate efficiency is worse than Turing+. Driver
  minimum for current Jellyfin (10.11+) is **520.56.06**; TrueNAS 25.10 ships
  **570.172.08**, so that floor is cleared with large margin.
- **Verify (point 5):** `docker exec -it jellyfin nvidia-smi` shows the GPU;
  `docker exec -it jellyfin jellyfin-ffmpeg -encoders 2>/dev/null | grep nvenc`
  shows `h264_nvenc`/`hevc_nvenc`; end-to-end proof is forcing a transcode and
  watching an NVENC session appear in `nvidia-smi`.

## The compose block to plan around

Modern Compose-Spec (what to write into `truenas/apps/jellyfin/compose.yaml`):

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    # ... volumes, ports, networks per repo conventions ...
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1                 # single P400. Mutually exclusive with device_ids.
              capabilities: [gpu]      # MANDATORY; deploy fails without it.
    # No runtime: key. No NVIDIA_* env vars — the linuxserver image sets them.
```

Pinning a specific GPU when multiple are present (not our case today, one P400):

```yaml
            - driver: nvidia
              device_ids: ['0']        # use instead of count; from `nvidia-smi`
              capabilities: [gpu]
```

> Field rules (Docker Compose spec): `capabilities` is **required**; `count`
> accepts an integer or the literal string `all`; `count` and `device_ids` are
> **mutually exclusive** — defining both errors on deploy.

<details>
<summary><b>Point 1 — compose passthrough syntax (full detail)</b></summary>

Two forms exist. **Modern is preferred; legacy is deprecated but still widely
seen in community guides.**

**Modern — `deploy.resources.reservations.devices` (Compose Spec v2, Docker
Compose ≥ 1.28 / Docker Engine ≥ 19.03):**

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all                  # or an int, or use device_ids
          capabilities: [gpu]
```

This is what **both** Jellyfin's own docs and Docker's own docs publish
today. The `deploy.devices` block maps to the same code path as `docker run
--gpus`. The `nvidia` runtime is selected automatically by the toolkit's
runtime hook when a `driver: nvidia` reservation is present — **you do not set
`runtime:` in modern Compose.**

**Legacy — `runtime: nvidia` + `NVIDIA_VISIBLE_DEVICES`:**

```yaml
runtime: nvidia
environment:
  - NVIDIA_VISIBLE_DEVICES=all        # or a GPU UUID / index
  - NVIDIA_DRIVER_CAPABILITIES=all    # or compute,utility,video
```

This is the `nvidia-docker2` era API. Still functional on systems configured
with the toolkit, but Docker Compose emits deprecation noise and the
`runtime:` key is ignored by some Compose versions when `deploy.devices` is
also present. **Avoid mixing the two** on one service.

**What each source currently recommends:**

| Source | Currently documents |
|---|---|
| Jellyfin docs (`jellyfin.org/.../nvidia`) | `runtime: nvidia` **and** the `deploy.resources.reservations.devices` block shown together. The doc text is slightly hybrid; the `deploy` block is the operative part. |
| linuxserver.io `jellyfin` image docs | Does **not** show a GPU block in its example compose. States you must (re)create the container with the **nvidia container runtime** (`--runtime=nvidia`) and directs you to the toolkit; says the image "automatically add[s] the necessary environment variable that will utilise all the features available on a GPU on the host" (i.e. it sets `NVIDIA_DRIVER_CAPABILITIES` for you — you do not). |
| Docker docs (`docs.docker.com/compose/how-tos/gpu-support`) | Modern `deploy.resources.reservations.devices` only. |

**Recommendation for our repo:** use the modern `deploy.resources.reservations.devices`
form, `count: 1` (single GPU), `capabilities: [gpu]`. Do not set `runtime:`,
do not set `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES` — let the
linuxserver image and the toolkit's runtime hook handle them.

**Device paths:** You do **not** manually mount `/dev/nvidia0`,
`/dev/nvidiactl`, or `/dev/dri/renderD128` for NVIDIA. The toolkit's runtime
injects them. (Manual `/dev/dri` device mapping is the **Intel/AMD** path and
is wrong for NVIDIA.) linuxserver.io confirms: "NVIDIA automatically mounts
the GPU and drivers from your host into the container."

</details>

<details>
<summary><b>Point 2 — NVIDIA Container Toolkit host-side dependency (full detail)</b></summary>

The `deploy.resources.reservations.devices` / `--gpus` path does **not** work
without host-side plumbing. Required, in order:

1. **NVIDIA driver on the host.** TrueNAS SCALE 25.10 ships
   **570.172.08** (open GPU kernel modules) and exposes driver install via the
   web UI — so the driver itself is present/managed for our box.
2. **`nvidia-container-toolkit` package** (provides `nvidia-ctk` and the
   `nvidia` container runtime). Canonical install (Debian/Ubuntu):
   ```bash
   curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
     | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit.gpg
   curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
     | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit.gpg] https://#g' \
     | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
   sudo apt-get update
   sudo apt-get install -y nvidia-container-toolkit
   ```
3. **Register the runtime with dockerd:**
   ```bash
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```
   `nvidia-ctk runtime configure --runtime=docker` modifies
   `/etc/docker/daemon.json` to register `nvidia` as a **named runtime** under
   `runtimes` (it does **not** force `default-runtime: nvidia`). The produced
   config is equivalent to:
   ```json
   {
     "runtimes": {
       "nvidia": {
         "path": "nvidia-container-runtime",
         "runtimeArgs": []
       }
     }
   }
   ```
4. **Verify** the host plumbing (run on the host, not in the Jellyfin
   container):
   ```bash
   docker run --rm --gpus all ubuntu nvidia-smi
   ```

**Toolkit supports** docker, containerd, CRI-O, Podman. For Podman NVIDIA
recommends CDI instead (`nvidia-ctk cdi generate`); not relevant to us on
TrueNAS Docker.

**CRITICAL — TrueNAS host availability (the unknown that gates everything):**

- `nvidia-container-toolkit` is **not installed by default** on the TrueNAS
  SCALE appliance host in a form usable by custom Compose containers.
- `apt` is **disabled by default** on the TrueNAS host shell — "TrueNAS is an
  appliance OS." Anything installed via `apt` directly on the host is
  **non-persistent** (wiped on reboot) unless committed via a custom
  middleware/overlay mechanism iX does not officially support for this.
- TrueNAS has an **internal** mechanism that exposes the GPU to its own
  first-party Apps (using the web-UI driver), but that path is not available to
  a Dockhand-managed Compose stack, which talks to the raw `docker.sock`.
- Community workarounds people use on TrueNAS for custom-Docker GPU work:
  - install the toolkit inside a **Jailmaker nspawn jail** that itself runs
    Docker, and run Jellyfin there; or
  - PCIe-passthrough the GPU into a **Linux VM** and run the stack inside it;
    or
  - run `install-dev-tools` on the host and `apt install` (non-persistent).

This is exactly the kind of thing that **cannot be resolved from docs alone**
— it requires inspecting the live host. **Handed to ticket #12** ("verify host
prerequisites") to confirm: (a) is `nvidia-ctk` / `nvidia-container-runtime`
present on the TrueNAS 25.10 host at all, (b) is `/etc/docker/daemon.json`
registering the `nvidia` runtime, (c) does `docker run --rm --gpus all ...`
succeed on the host. If any of those is "no," the topology decision must
choose between the jail/VM path or abandoning GPU transcoding — Dockhand
cannot paper over a missing host runtime.

</details>

<details>
<summary><b>Point 3 — Dockhand compatibility (the gitops risk, full detail)</b></summary>

**What Dockhand's manual (`dockhand.pro/manual`) does say:**

- It is a wrapper around the `docker compose` CLI, not a custom YAML engine.
  > "Dockhand reads the docker-compose.yml file from its own local filesystem
  > ... and sends the instructions directly to the remote Docker daemon."
- Git Stack = repo + branch + **compose file path** (default
  `docker-compose.yml`) + optional context directory; the whole context dir is
  cloned so co-located files travel with it.
- Per-stack deploy options exist as **flags Dockhand adds to `docker compose
  up`**: `--build`, disable build cache, re-pull images, force redeployment.
  (e.g. "Adds `--build` to `docker compose up`.")

**What the manual does NOT say (and this is the risk):**

- **No mention** of GPU, NVIDIA, CDI, `runtime`, `deploy.resources`,
  `reservations`, or `devices` anywhere.
- **No documented list** of supported/unsupported compose keys.
- **No documented sanitization, validation, or stripping** of unknown keys.
- **No UI control** to choose a docker runtime per service or per stack. The
  word "runtime" in the manual refers only to alternative container engines
  (Podman), not to docker runtimes like `nvidia`/`runsc`.

**Inference (treat as plausible but unverified):** Because Dockhand is
described as reading the file and shelling to `docker compose` rather than
parsing/re-serializing it, **unknown Compose-Spec keys should pass through
verbatim** and be honored by Docker Compose itself — i.e.
`deploy.resources.reservations.devices` should work. This is consistent with
how Compose-Spec wrappers (Portainer, Dockge) generally behave.

**What still must be validated on the box before trusting it in gitops:**

1. After Deploy, does the container actually come up with the GPU attached?
   (`docker exec jellyfin nvidia-smi` — see point 5.)
2. Does Dockhand's stack editor **save** the `deploy.resources.reservations.devices`
   block unchanged after a round-trip (open the stack in the UI, confirm the
   YAML is intact)? This catches any silent re-serialization.
3. Does the diff-gated auto-sync / webhook re-apply it consistently?

If (1) fails but the host toolkit (point 2 / #12) is confirmed working, the
fallback is the **legacy form** (`runtime: nvidia` + `NVIDIA_VISIBLE_DEVICES`)
which some Compose-on-trueNAS users report as more reliably honored end-to-end
— at the cost of deprecation warnings and the `runtime:` key.

</details>

<details>
<summary><b>Point 4 — Jellyfin-side NVENC enablement (full detail)</b></summary>

**UI path:** Dashboard (top-left hamburger) → **Playback → Transcoding** →
**Hardware acceleration: NVIDIA NVENC**. Then enable the codec checkboxes you
want: **H.264**, **HEVC** (and MPEG2/VC1/VP9 decode if useful). NVENC is
**not** auto-enabled just because the GPU is present — you must select it.

**P400 (Pascal GP107) capability — from NVIDIA's own support matrix:**

| Capability | P400 |
|---|---|
| H.264 (AVC) encode — YUV 4:2:0 / 4:4:4 / Lossless | YES |
| HEVC (H.265) encode — 4K YUV 4:2:0 / 4:4:4 / 8K / 10-bit | YES |
| **HEVC B-frame encode** | **NO** ← the load-bearing caveat |
| AV1 encode | NO (needs Ada Lovelace) |
| H.264 decode (NVDEC) | YES (8-bit 4:2:0) |
| HEVC decode (NVDEC) | YES (8/10-bit 4:2:0) |
| Max concurrent NVENC sessions (Quadro) | 8 (per matrix) |
| NVENC generation (matrix label) | "6th Gen" (note: NVIDIA renumbered generations in this matrix; the die is Pascal, single NVENC engine) |

**P400-specific caveats to flag:**

- **No HEVC B-frames.** Quality at a given bitrate for HEVC transcodes is
  worse than Turing+ (RTX 20xx) which first got HEVC B-frame support. H.264
  B-frame support is not separately broken out in the matrix but H.264 NVENC
  B-frames predate Pascal, so H.264 B-frames are fine.
- **Single NVENC engine** → transcodes serialize through one encoder. Fine for
  a homelab; will bottleneck on 3+ simultaneous transcodes.
- **Session limits:** NVIDIA's matrix lists 8 for Quadro (no artificial
  consumer cap). Historically GeForce consumer NVENC was capped at ~3 (later
  raised to 5 on driver ~545+, Oct 2023) — **this is community-reported; I
  could not re-verify the exact driver version from primary sources this
  pass.** For a Quadro card this is moot; flagging only for completeness.
- **Driver minimum:** Jellyfin 10.11 requires driver ≥ **520.56.06** (Linux) /
  522.25 (Windows). TrueNAS 25.10 ships **570.172.08**, so the floor is
  cleared by a wide margin.
- **No license needed.** NVENC on Quadro is unlocked; unlike consumer GeForce
  there is no driver-enforced session cap to work around.

**Jellyfin config tip:** also enable **HDR tone-mapping** if you want HDR→SDR
transcodes (the linuxserver image ships the `nvidia.icd` fix for this since
2021).

</details>

<details>
<summary><b>Point 5 — verification command (full detail)</b></summary>

Inside the **running Jellyfin container** (after Deploy), in order of strength:

```bash
# 1. GPU is visible to the container at all (proves passthrough + runtime).
docker exec -it jellyfin nvidia-smi
# Expect: a Quadro P400 row, driver 570.172.08, CUDA version.

# 2. Jellyfin's ffmpeg sees the NVENC encoders (proves userspace libs).
docker exec -it jellyfin jellyfin-ffmpeg -encoders 2>/dev/null | grep nvenc
# Expect: V..... h264_nvenc     NVIDIA NVENC H.264 encoder
#         V..... hevc_nvenc     NVIDIA NVENC hevc encoder

# 3. End-to-end: force a transcode (play something, lower bitrate/res in the
#    client) and watch NVENC engage on the host while it runs:
nvidia-smi -l 1            # run on the HOST; look for the ffmpeg/jellyfin
                           # process under "Processes" and NVENC utilization.
```

Optional deeper checks:

```bash
# ffmpeg hardware-accelerators list (should include cuda)
docker exec -it jellyfin jellyfin-ffmpeg -hwaccels

# A standalone probe encode (no Jellyfin involvement) — strongest single-shot
# proof that the whole stack (driver → runtime → libs) works:
docker exec -it jellyfin jellyfin-ffmpeg -f lavfi -i testsrc=duration=5:size=1280x720:rate=30 \
  -c:v h264_nvenc -f null -
```

Jellyfin also logs the result of its own NVENC capability probe at startup;
grep the Jellyfin log for `nvenc` to see which encoders it registered.

If (1) fails with `command not found` or `no devices were found`, the
passthrough/runtime is broken — go back to point 2 / #12. If (1) works but (2)
shows no `h264_nvenc`, the image's NVIDIA libs are not engaged (usually a
missing `NVIDIA_DRIVER_CAPABILITIES=video` — but the linuxserver image sets
this, so this branch is unlikely for us).

</details>

## Confidence notes

- **Doc-backed, high confidence:** the modern compose syntax and field
  semantics (Docker + Jellyfin + linuxserver docs); the toolkit install +
  `nvidia-ctk runtime configure` + `daemon.json` named-runtime behavior (NVIDIA
  Container Toolkit install guide); the P400 NVENC encode/decode matrix
  including **HEVC B-frame = NO** (NVIDIA's own GPU support matrix); driver
  minimum 520.56.06 for Jellyfin 10.11 (Jellyfin docs); TrueNAS 25.10 driver
  570.172.08 (TrueNAS forum).
- **Inference, not doc-stated (flag):** that Dockhand passes unknown compose
  keys (incl. `deploy.resources.reservations.devices`) through to `docker
  compose` verbatim. Consistent with how Dockhand is described (a wrapper, not
  a parser) but **not guaranteed by dockhand.pro** — must be validated on-box.
- **Host-state, cannot resolve from docs (flag → #12):** whether
  `nvidia-container-toolkit` / the `nvidia` runtime is actually installed and
  registered on the TrueNAS 25.10 host for custom-Docker use. Docs say it is
  not present by default and `apt` is disabled; only the live host can confirm.
- **Community-sourced (lower confidence):** the consumer NVENC session-limit
  history (3→5 on driver ~545, Oct 2023). Not relevant for a Quadro card;
  included only for completeness.

## Sources

- [Jellyfin — NVIDIA GPU hardware acceleration](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/nvidia/)
- [Jellyfin — Hardware acceleration overview](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)
- [linuxserver.io — docker-jellyfin docs](https://docs.linuxserver.io/images/docker-jellyfin/)
- [Docker — Run Compose services with GPU access](https://docs.docker.com/compose/how-tos/gpu-support/)
- [Compose deploy spec — devices](https://docs.docker.com/reference/compose-file/deploy/)
- [NVIDIA Container Toolkit — install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [NVIDIA Container Toolkit — user guide (1.14.1)](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.14.1/user-guide.html)
- [NVIDIA — Video Encode and Decode GPU Support Matrix](https://developer.nvidia.com/video-encode-and-decode-gpu-support-matrix-new)
- [NVIDIA — NVENC Video Codec SDK application note](https://docs.nvidia.com/video-technologies/video-codec-sdk/13.0/nvenc-application-note/index.html)
- [Dockhand — User Manual](https://dockhand.pro/manual/) · [dockhand.pro](https://dockhand.pro) · [github.com/Finsys/dockhand](https://github.com/Finsys/dockhand)
- [TrueNAS forum — 25.10.2.1 Goldeye Docker NVIDIA driver issues](https://forums.truenas.com/t/25-10-2-1-goldeye-docker-nvidia-driver-issues/64220) (driver 570.172.08 confirmation)
- [TrueNAS forum — Electric Eel NVIDIA GPU passthrough support](https://forums.truenas.com/t/electric-eel-nvidia-gpu-passthrough-support/11797) (apt disabled, appliance model)
- [Repo briefing — `research/dockhand-capabilities.md`](./dockhand-capabilities.md)
