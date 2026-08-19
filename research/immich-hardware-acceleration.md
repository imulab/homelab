# Immich hardware acceleration on the Arc A380 (ML + video) and CPU-only sizing

> Resolves wayfinder ticket: can Immich productively share the Intel Arc A380 with Jellyfin, and what does the stack cost in CPU/RAM if it cannot? Compiled 2026-08-19 via web research against Immich docs (viewed 2026-08-19; current release v3.1.0, 2026-07-29), the Immich repo, OpenVINO/Intel docs, and community threads. Builds on [host-driver-i915-vs-xe-and-guc-huc-firmware.md](./host-driver-i915-vs-xe-and-guc-huc-firmware.md) (A380 = Alchemist, ACM-G11, PCI 8086:56a5, 128 EUs, i915 + GuC submission mandatory, HuC for HEVC) and [container-passthrough-dev-dri-to-jellyfin.md](./container-passthrough-dev-dri-to-jellyfin.md) (render-node + `group_add` pattern already proven on this box for Jellyfin QSV).

## TL;DR

- **Yes, the A380 is a supported OpenVINO target for Immich's ML**, and QSV for Immich's ffmpeg transcodes is documented. The host-side prerequisites (i915, GuC/HuC, `/dev/dri/renderD128`, render GID via `group_add`) are already solved on this box per the two research docs above; Immich needs the same passthrough, not new host work.
- **But ML-on-GPU has a real kernel-version coupling.** One well-documented A380 case (Immich discussion #24276): OpenVINO saw only `['CPU']` on kernel 6.8 while QSV transcoding worked fine; a kernel upgrade to 6.17 made `GPU` appear. OpenVINO's own docs say kernel 6.2+ is "recommended" for Arc, but the observed floor for this exact card + in-container compute runtime is higher than that. TrueNAS SCALE 24.10 is kernel 6.6.44, 25.10 is 6.12. The result on either is **undetermined**: this is the single biggest day-one bring-up risk of GPU ML.
- **QSV for Immich video is low-risk but low-value.** Only the video *transcode* path accelerates (encoding by default; decode optional). Video thumbnail generation does **not** use hardware acceleration at all (open feature request, discussion #11222), so the heaviest video cost during a library import stays on CPU regardless.
- **Sharing the A380 with Jellyfin is architecturally fine**: QSV uses the fixed-function media engines, OpenVINO uses the EU/shader fabric; they contend mainly for VRAM and memory bandwidth, not the same execution blocks. One community author runs Immich OpenVINO ML + Immich QSV transcodes + Jellyfin QSV on a single 80EU iGPU with no observed contention (300k photos embedded in under a night).
- **CPU-only verdict: comfortable for a family library of 20-100k assets.** Immich's floor is 2 cores / 6 GB RAM (recommended 4 cores / 8 GB), the whole stack idles around 2 GB, and the one-time initial ML pass costs roughly 0.5-7 CPU-days depending on library size and CPU class. Steady-state background load for a family (tens to hundreds of new photos a day) is trivially absorbed.
- **Recommendation for the decision ticket:** bring Immich up CPU-only on day one; treat OpenVINO-on-A380 as a phase-2 experiment gated on verifying GPU detection in the ML logs; enable QSV transcodes only if video transcode CPU load is ever observed to matter. Sizing conclusions scale with the box's actual CPU model and RAM, which the decision ticket still confirms.

## 1. ML acceleration: CPU (default) vs OpenVINO on x86

### What the options are

Immich's ML container (`immich-machine-learning`) runs ONNX Runtime. Backend is selected by **image tag suffix**, not a config flag (docs: ML hardware acceleration, viewed 2026-08-19): the default image is CPU; `-openvino` is the Intel backend (others: `-cuda`, `-rocm`, `-armnn`, `-rknn`). The feature is officially labeled experimental. Enabling it:

```yaml
services:
  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-openvino
    # extends: file: hwaccel.ml.yml, service: openvino   (or inline it)
```

The stock `docker/hwaccel.ml.yml` `openvino` block in the repo (viewed 2026-08-19) is exactly:

```yaml
  openvino:
    device_cgroup_rules:
      - 'c 189:* rmw'
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /dev/bus/usb:/dev/bus/usb
```

(`c 189` is the USB bus char device, a leftover for Intel NCS neural sticks; harmless.) Note Immich's own file binds the whole `/dev/dri` directory, while our Jellyfin doc recommends binding `renderD128` (+ `card0` if OpenCL) explicitly. Both work; the explicit form is safer on a host that ever grows a second GPU. As with Jellyfin, the container needs the host render GID via `group_add` if the node is `0660` (our passthrough doc: check with `stat -c '%g' /dev/dri/renderD128`; 107 observed on SCALE, not 104).

**In-container runtime:** no extra setup is needed, the image ships its own Intel GPU userspace. The `prod-openvino` stage of `machine-learning/Dockerfile` installs `ocl-icd-libopencl1`, `intel-igc-core/opencl 2.36.3`, `intel-opencl-icd 26.22.38646.4` (+ a `legacy1` 24.35 build), older IGC 1.0.17537.24, and `libigdgmm12 22.10.0`. Notably it does **not** install `level-zero`, so OpenVINO reaches the GPU through OpenCL here. The Python side is `onnxruntime-openvino>=1.24.1` (machine-learning/pyproject.toml). The practical consequence: GPU detection depends on the **compute runtime debs pinned in the image** cooperating with the **host kernel's i915**, which is exactly what bit the A380 case below.

**Relevant env vars** (docs: environment variables): `MACHINE_LEARNING_DEVICE_IDS` (default `0`; comma-separated GPU device IDs, needs `MACHINE_LEARNING_WORKERS>1`, one device per worker round-robin), `MACHINE_LEARNING_WORKERS` (default `1`; each worker duplicates all models in memory, docs advise against raising), `MACHINE_LEARNING_REQUEST_THREADS` (default: number of CPU cores), `MACHINE_LEARNING_MODEL_INTRA_OP_THREADS` (default `2`). No env var turns OpenVINO on or off: the tag does that. When the `-openvino` image runs and a GPU is visible, it is used automatically; otherwise the log shows `No GPU device found in OpenVINO. Falling back to CPU.` (observed verbatim in #24276).

### Is Arc (ACM-G11) supported?

Yes, at every layer, in principle:

- Immich docs: OpenVINO supports "Intel GPUs such as Iris Xe and Arc".
- OpenVINO GPU plugin README: "Intel HD Graphics, Intel Iris Graphics, and Intel Arc Graphics" (Alchemist/Xe-HPG is in the supported set).
- Intel compute-runtime platform table: Alchemist, OpenCL 3.0 / Level Zero 1.17.
- OpenVINO GPU configuration docs: on Linux, GPU inference needs the OpenCL runtime packages (`intel-opencl-icd`, `intel-level-zero-gpu`, `level-zero`, `ocl-icd-libopencl1`) and the user in the `render` group (or root); "For Arc GPU, kernel 6.2 or higher is recommended."

### Expected speedup and VRAM appetite

There is no official benchmark. What exists:

- **Immich docs (qualitative):** acceleration reduces CPU load; bigger models gain more *if you have the VRAM for them*; raising concurrency raises VRAM; every GPU in a multi-GPU setup must hold all models (no splitting); expect **higher RAM usage with OpenVINO than CPU**; iGPUs are likelier to hit issues than discrete cards; ensure "the server's kernel version is new enough".
- **Negative datapoint (iGPU, 2024, discussion #8104):** on an i5-10500 with UHD 630, OpenVINO showed *no* throughput gain over CPU (~2 assets/5 s at 2 concurrent jobs on either backend, with a large `nllb-clip-large-siglip` model). Maintainer mertalev attributed it to a weak iGPU, no input batching to the GPU yet ("OpenVINO may be more sensitive to that than CUDA"), and one-time model-compilation overhead that can take minutes.
- **Positive datapoint (iGPU, Nov 2025, t0saki.com):** on an i5-13500H's 80EU Xe iGPU, OpenVINO embedded **300k photos' CLIP in less than one night** (order of 7 assets/s sustained, vs ~0.4 assets/s in the CPU datapoint above, albeit different models), and ran the buffalo_l facial detection/recognition pass over 300k in tens of minutes. VRAM (shared iGPU memory) measured at ~6.8 GB peak with a deliberately large stack (large CLIP + inswapper + buffalo_l at high concurrency), and the author explicitly warns VRAM exhaustion is easy if you pick big models plus concurrency; peak RAM of the CPU-only run on the same box was ~3.6 GB.
- **Order-of-magnitude for defaults:** Immich's default CLIP is the small ViT-B-32 and default facial recognition buffalo_l (FAQ discusses dropping to buffalo_s to save CPU); these are far below the large-model stacks above. Treat ~1-3 GB VRAM for the default model set as a reasonable planning number (my estimate, not a doc figure). The A380's 6 GB GDDR6 fits that with headroom left for QSV sessions.

**Speedup summary:** on a discrete Arc part expect a meaningful multiple (the two datapoints bracket roughly 1x on a weak iGPU to ~15-20x on a recent iGPU with large models). For Immich's *default small* models the gain is smaller and the docs' honest framing is "reduces CPU load", not "N times faster".

### The kernel-version trap (the load-bearing caveat)

Immich discussion #24276 (ptr727, Nov-Dec 2025), an **Arc A380 specifically**:

- Proxmox, kernel `6.8.12-16-pve`, Docker on the host, `release-openvino` image, `/dev/dri` exposed, render/video groups added, `user: root` tried, cgroup rules tried, `/dev/bus/usb` mounted. QSV hardware transcoding worked on the same GPU the whole time. OpenVINO still logged `Available OpenVINO devices: ['CPU']` and `No GPU device found in OpenVINO. Falling back to CPU.` The stock Intel OpenVINO toolkit container failed the same way.
- **Fix: upgrade the kernel 6.8 -> 6.17.** After that, `Available OpenVINO devices: ['CPU', 'GPU']`. Author's own conclusion: "There must be some minimum kernel version required for this to work?" No maintainer or Intel doc pins the exact floor.

Implication for this box: SCALE 24.10 (Electric Eel) runs 6.6.44 and 25.10 (Goldey) runs 6.12 LTS (kernel table in the host-driver doc). Both sit **below** the 6.17 that fixed the one documented A380 case and **at or above** the 6.2 that OpenVINO docs "recommend". Nobody in the public record confirms A380 OpenVINO compute on 6.6 or 6.12. Since i915 kernel and the image-pinned compute runtime must agree, and the image pins a very new runtime (26.22.x), an older host kernel is the plausible failure axis. The only way to know is to run the `-openvino` image and read the log line.

Also budget for regressions: issue #25830 documents v2.5.2 breaking OpenVINO ML on a Jasper Lake iGPU (`clCreateSubBuffer, error code: -30`) after months of working; v2.4+ ML OCR memory-leak reports exist (#23462, #26739). The OpenVINO path needs re-verification after Immich updates; pin the ML image tag.

## 2. Video: QSV on Arc for Immich

Documented and supported (docs: hardware transcoding, viewed 2026-08-19):

- **Config surface:** `docker/hwaccel.transcoding.yml` `quicksync` block is just `devices: - /dev/dri:/dev/dri` on the `immich-server` container; then Admin UI -> Video transcoding -> hardware acceleration = **Intel Quick Sync**. Config-file equivalent: `ffmpeg.accel: "qsv"`, optional `accelDecode: true`. There is no env var for this on Linux (only Unraid's NVIDIA template uses `NVIDIA_VISIBLE_DEVICES`).
- **What it buys:** hardware *encoding* of transcodes (the one-time and on-demand video transcode jobs). Hardware decoding is opt-in ("end-to-end acceleration", "may not work on all videos"). Tone-mapping is a separate path. Docs warn output files are "significantly larger... typically with lower quality" than software transcodes; treat it as experimental.
- **What it does NOT buy: video thumbnails.** Thumbnail/preview extraction runs plain software ffmpeg; it does not go through the hwaccel path (open feature request #11222; maintainer alextran1502: "it just hasn't been implemented", mertalev: the ffmpeg-command handling would need refactoring). Community workaround: mount a wrapper script over `/usr/sbin/ffmpeg` that injects `-hwaccel` for thumbnail jobs (hacky, video-only). So during initial import, video thumbnails still burn CPU either way.
- **On the A380 specifically:** the QSV path is the same media-engine path (VE box + GuC/HuC) Jellyfin already exercises on this box, per the host-driver and passthrough docs. Docs' generational caveats (VP9 needs 9th gen+, 11th-gen low-power mode, kernel 5.15) are about iGPUs and do not apply to a DG2 card on a 6.6/6.12 kernel.

Net: enabling QSV for Immich later is a two-line compose change plus an admin toggle, riding infrastructure already proven by Jellyfin. But since Immich transcodes are batch background jobs rather than latency-sensitive streams, and thumbnails are unaffected, CPU transcoding is usually acceptable at family scale.

## 3. Sharing the A380 between Immich and Jellyfin

### Architecture: mostly different engines

- **Jellyfin QSV** runs on the A380's fixed-function media engines (VEBOX/VDBOX, HuC for HEVC bitstream, per the host-driver doc).
- **Immich OpenVINO ML** runs on the EU/shader fabric through OpenCL (compute-runtime/NEO), per the compute-runtime and OpenVINO GPU plugin docs.
- These are distinct hardware blocks; the shared resources are **VRAM (6 GB) and memory bandwidth**. On DG2 every context is scheduled by the GuC (GuC submission is the only path on this silicon, per the host-driver doc); there is no user-facing per-container QoS or fair-share knob.

### What is actually observed

- **t0saki (Nov 2025):** one 80EU iGPU simultaneously serving Immich OpenVINO ML, Immich QSV transcoding, and Jellyfin transcoding, with GPU maxing at 100% during ML passes and "no impact on other services" reported. This is an iGPU (shared system memory), but it is the same engine-separation argument; the A380 has more compute and dedicated VRAM.
- **jonathangazeley.com (Feb 2025):** runs Jellyfin + Immich (QSV and OpenVINO) against one iGPU on Kubernetes. The GPU-sharing problem he hits is **orchestrator-level**: the Intel device plugin makes `gpu.intel.com/i915` exclusive unless configured with `shared-dev-num`. **Irrelevant under plain Docker/Dockhand**: there is no device plugin, both containers just open `/dev/dri/renderD128` directly, and the kernel driver arbitrates. (Also confirms Immich ML needs generous startup time while models preload, ~120 s probe.)
- **Our own Jellyfin doc** already establishes the A380 comfortably serving 10+ concurrent 4K QSV transcodes (r/truenas) on this class of host, i.e. the media engines have slack.
- **richay.au LXC guide:** standard practice is passing the same Arc render node into multiple containers; no contention limits needed, only GID mapping hygiene.

### Failure modes to expect (ranked by evidence)

1. **Permissions, the boring one:** wrong/missing render GID in `group_add` produces silent CPU fallback (OpenVINO) or failed encoder init (QSV). Solved by the `stat`-then-hardcode pattern in our passthrough doc.
2. **Kernel vs compute-runtime mismatch:** GPU invisible to OpenVINO while QSV works (#24276). Not a sharing problem per se, but it masquerades as one.
3. **VRAM exhaustion:** no hard per-container VRAM allocation or limit exists on i915 for these userspaces; if Immich's models plus several QSV sessions overshoot 6 GB, allocations fail (encoder session errors, OpenCL context creation failures). Mitigation: keep default small models, `MACHINE_LEARNING_WORKERS=1`, moderate job concurrency, and remember QSV sessions release VRAM when streams end. Evidence for the exact symptoms is thin/anecdotal; the mechanism (no isolation) is well established. Jellyfin bursts (HDR tonemapping) are the heaviest QSV VRAM consumers.
4. **Version regressions in Immich's OpenVINO path** (#25830): re-verify after upgrades.

## 4. CPU-only sizing for a family deployment

All numbers below assume the *default* model set (CLIP ViT-B-32, buffalo_l) unless noted. The decision ticket still confirms this box's CPU model and RAM; where that matters it is flagged.

### Floors and overheads (doc-backed, requirements + FAQ pages)

| Item | Number | Note |
|---|---|---|
| CPU minimum / recommended | 2 cores / 4 cores | docs floor |
| RAM minimum / recommended | 6 GB / 8 GB | whole stack, all containers |
| On 4 GB hosts | run with ML disabled | documented escape hatch |
| Stack idle RAM | ~2 GB | community (r/immich, Docker Desktop); ML keeps models resident |
| ML container alone | ~1-2 GB resident | models loaded; each extra worker duplicates them |
| Thumbnails + transcodes storage | +10-20% of library | docs |
| Postgres | 1-3 GB DB files; give it >=2 GB RAM if limiting | docs; DB on local SSD, never network share |
| Redis | small, job queue only | no doc number; plan ~100 MB (estimate) |
| CPU baseline (amd64) | x86-64-v2 since v3 | pre-2013 CPUs capped at v2.7.5 |

### Initial library scan, order of magnitude per 10k photos

The initial pass runs per asset: thumbnail jobs (3 per photo: thumbhash, WebP preview, JPEG thumbnail), CLIP embedding, then face detection + recognition. Measured datapoints:

- CPU CLIP, i5-10500, large model, 2-4 concurrent jobs: ~0.4 assets/s (discussion #8104). That is ~7 h per 10k photos for CLIP alone; face detection/recognition adds a pass of comparable or greater cost (two models, FAQ suggests dropping to buffalo_s to halve it).
- Whole-pipeline anecdote: ~200k photos estimated ~55 h end to end on CPU (r/immich, ~1 asset/s).
- Same workload class on a recent 80EU iGPU via OpenVINO: ~7 assets/s, 300k in a night (t0saki).

**Planning number for CPU-only:** roughly 0.5-2 assets/s per 2-4 job slots on a modern desktop-class 4-8 core CPU. So 20k assets ≈ 3-12 h, 50k ≈ 7-30 h, 100k ≈ 0.5-2.5 days of mostly unattended background churning. Scale up/down with the box's actual core count and generation (decision-ticket dependency); a NAS-class 4-core part sits at the slow end, a 12th-gen+ 8-core near the fast end.

### Steady-state background load

After the initial scan, a family generating tens to hundreds of new photos per day adds minutes of ML work per day; phone backups trickle in and the job queue drains on 1-2 cores overnight. Video thumbnails during import are the only bursty CPU item (unacceleratable today, see section 2). Job concurrency of 2-3 "can probably max the CPU" per the FAQ; set it to 1-2 if Immich shares the box with a Jellyfin transcode storm.

### Co-tenancy on this box

Jellyfin CPU use is near zero when it transcodes on the A380; Dashy, Caddy, Dockhand are all sub-100 MB, sub-0.5-core services. The real co-tenant question is RAM: Immich ~4-6 GB worst case under load (server + ML + Postgres + Redis) on top of everything else. On a 16 GB box that fits; on 32 GB it is roomy. If the box turns out to be 8 GB total, Immich works but leaves little headroom for ZFS and other stacks; 6 GB is the documented floor.

## 5. Recommendation hooks for the decision ticket

1. **CPU-only is good enough for a family library (20-100k assets).** Verdict: comfortable. The cost is a one-time initial scan of hours to a couple of days, and ~2-6 GB of RAM; steady-state load is negligible. No ML feature is lost, only latency on the initial index.
2. **Day-one GPU sharing adds real bring-up risk for ML, not for video.** OpenVINO-on-A380 has one documented hard failure on kernel 6.8 (fixed at 6.17) and our host is on 6.6.44 or 6.12 depending on SCALE version; outcome undetermined until tested. Debugging "GPU not found, silent CPU fallback" is a poor day-one activity.
3. **Suggested sequencing:** deploy CPU-only; let the initial scan run; then, as a phase-2 experiment, swap the ML image to `-openvino` with the `renderD128` + `group_add` block (same pattern as Jellyfin), verify the log shows `Available OpenVINO devices: ['CPU','GPU']` and watch `intel_gpu_top` during a CLIP job; roll back to the CPU image if not. QSV for Immich transcodes can be enabled any time with two compose lines and a dashboard toggle, but buy little at family scale.
4. **If GPU ML is adopted:** keep default small models, `MACHINE_LEARNING_WORKERS=1`, moderate concurrency, pin the ML image tag, and re-verify after every Immich upgrade (#25830 precedent).
5. **Sizing dependencies on the decision ticket's CPU/RAM confirmation:** initial-scan wall-clock scales ~linearly with effective cores and CPU generation; the RAM verdict flips from "comfortable" to "tight" below 16 GB total. Everything else here is host-independent.

## Confidence notes

- **Doc-backed (Immich docs/repo, viewed 2026-08-19, release v3.1.0):** ML backends selected by image tag suffix; `-openvino` tag; `hwaccel.ml.yml` openvino block (`/dev/dri`, cgroup 189, `/dev/bus/usb`); ML image ships its own Intel compute runtime debs (IGC 2.36.3, intel-opencl-icd 26.22.38646.4, libigdgmm12), no level-zero; `onnxruntime-openvino>=1.24.1`; `MACHINE_LEARNING_DEVICE_IDS=0`, `MACHINE_LEARNING_WORKERS=1` defaults; OpenVINO supports "Iris Xe and Arc"; higher RAM with OpenVINO; kernel "new enough" warning; bigger models need more VRAM; QSV config surface (`/dev/dri:/dev/dri` + admin UI / `ffmpeg.accel: "qsv"`, `accelDecode`); encode-only by default; larger/lower-quality hw transcode output; video thumbnails not hw-accelerated (#11222 maintainer statements); requirements (2/4 cores, 6/8 GB RAM, x86-64-v2, DB 1-3 GB, +10-20% thumbnails); FAQ knobs (buffalo_s, concurrency 2-3, transcode threads).
- **Doc-backed (Intel/OpenVINO):** GPU plugin README lists Arc; compute-runtime table lists Alchemist (OpenCL 3.0, Level Zero 1.17); GPU config page requires OpenCL runtime packages + render group and "kernel 6.2 or higher recommended" for Arc.
- **Community-sourced:** A380 OpenVINO invisible on kernel 6.8, fixed on 6.17, while QSV worked (discussion #24276, detailed and credible); no CLIP speedup on UHD 630 + maintainer batching explanation (discussion #8104); 300k photos/night, 6.8 GB VRAM, 3.6 GB CPU RAM, shared with Jellyfin without contention (t0saki.com, Nov 2025); ~55 h for 200k photos on CPU (r/immich, search-surfaced, thread not directly readable); stack idle ~2 GB (r/immich); k8s device-plugin exclusivity and 120 s ML startup probe (jonathangazeley.com); v2.5.2 OpenVINO breakage (issue #25830); ML OCR memory-leak reports (#23462, #26739); r/immich Arc ML thread exists and is positive but was not directly readable, low weight.
- **Undetermined from public docs:** whether OpenVINO compute works on the A380 under kernel 6.6.44 (SCALE 24.10) or 6.12 (SCALE 25.10); the true minimum kernel for the image's pinned compute runtime; exact VRAM footprint of Immich's default model set on Arc (my 1-3 GB is an estimate); whether Immich hw transcode ever benefits video thumbnails (no, per #11222, but watch for implementation); concrete symptoms of A380 VRAM oversubscription across containers (mechanism clear, anecdotes thin); the Intel community "Arc A380 not detected in Docker" thread corroborates the failure mode but was fetch-blocked (403), cited by title only.

## Sources

[Immich docs: ML hardware acceleration](https://docs.immich.app/features/ml-hardware-acceleration) · [Immich docs: hardware transcoding](https://docs.immich.app/features/hardware-transcoding) · [Immich docs: requirements](https://docs.immich.app/install/requirements) · [Immich docs: FAQ](https://docs.immich.app/FAQ) · [Immich docs: environment variables](https://docs.immich.app/install/environment-variables) · [Immich repo: docker/hwaccel.ml.yml](https://github.com/immich-app/immich/blob/main/docker/hwaccel.ml.yml) · [Immich repo: docker/hwaccel.transcoding.yml](https://github.com/immich-app/immich/blob/main/docker/hwaccel.transcoding.yml) · [Immich repo: machine-learning/Dockerfile](https://github.com/immich-app/immich/blob/main/machine-learning/Dockerfile) · [Immich repo: machine-learning/pyproject.toml](https://github.com/immich-app/immich/blob/main/machine-learning/pyproject.toml) · [Immich releases (v3.1.0, 2026-07-29)](https://github.com/immich-app/immich/releases/latest) · [Immich discussion #24276: OpenVINO on Arc A380, kernel fix](https://github.com/immich-app/immich/discussions/24276) · [Immich discussion #8104: is OpenVINO accelerating smart search](https://github.com/immich-app/immich/discussions/8104) · [Immich discussion #11222: hw-accelerated thumbnails feature request](https://github.com/immich-app/immich/discussions/11222) · [Immich issue #25830: v2.5.2 OpenVINO breakage](https://github.com/immich-app/immich/issues/25830) · [Immich issue #23462: ML OCR memory](https://github.com/immich-app/immich/issues/23462) · [OpenVINO docs: Intel GPU configuration](https://docs.openvino.ai/2025/get-started/install-openvino/configurations/configurations-intel-gpu.html) · [OpenVINO GPU plugin README](https://github.com/openvinotoolkit/openvino/blob/master/src/plugins/intel_gpu/README.md) · [intel/compute-runtime platform table](https://github.com/intel/compute-runtime) · [t0saki.com: Immich GPU acceleration (Nov 2025)](https://t0saki.com/en/posts/immich-gpu-acceleration/) · [jonathangazeley.com: Intel GPU acceleration on Kubernetes (Feb 2025)](https://jonathangazeley.com/2025/02/11/intel-gpu-acceleration-on-kubernetes/) · [richay.au: shared Intel Arc in LXC for Plex/Jellyfin](https://richay.au/proxmox-unprivelliged-lxc-with-shared-intel-arc-gpu-for-plex-or-jellyfin-transcoding/) · [r/immich: 200k photo upload duration](https://www.reddit.com/r/immich/comments/1uxdj3p/how_can_i_speed_up_the_process_of_uploading/) · [r/immich: stack idle RAM](https://www.reddit.com/r/immich/comments/1k4dkbp/why_is_immich_using_almost_2_gigs_of_ram_on_idle/) · [r/immich: Arc GPU smart search/face recog](https://www.reddit.com/r/immich/comments/1h0ph7t/smartsearch_facerecog_on_intel_arc_gpu/) · [Intel community: Arc A380 not detected in Docker (title only, fetch-blocked)](https://community.intel.com/t5/Intel-Distribution-of-OpenVINO/Arc-A380-not-detected-in-Docker-on-Linux-fallback-to-CPU-why/m-p/1728412) · [host-driver-i915-vs-xe-and-guc-huc-firmware.md](./host-driver-i915-vs-xe-and-guc-huc-firmware.md) · [container-passthrough-dev-dri-to-jellyfin.md](./container-passthrough-dev-dri-to-jellyfin.md)
