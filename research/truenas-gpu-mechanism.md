# How the TrueNAS curated Jellyfin app exposes the GPU, and whether Dockhand can reuse it

> Resolves wayfinder ticket #15. Compiled 2026-08-07 via `/research` probe.
> Scope: how the TrueNAS SCALE 25.10 ("Goldeye") curated **Jellyfin** app gets
> GPU access, and whether a **Dockhand-managed Compose** container on the same
> host can reuse that mechanism. Research only — no host access, no config
> edits. Cross-references the #10 findings (`jellyfin-gpu-passthrough.md`);
> one of those findings is **reversed** here (see "Pascal reversal" below).

## TL;DR

- **Point 1 — platform (high confidence):** TrueNAS SCALE has been **Docker-based
  since 24.10 "Electric Eel"**; 25.10 "Goldeye" continues the Docker model
  (Docker Engine 28.3.1). The curated Jellyfin app is a plain Docker Compose
  stack rendered from iX's catalog, **not** a k3s pod. iX release notes confirm
  both halves.

- **Point 2 — the exact GPU mechanism (highest confidence, read in source):**
  The curated app uses the **modern Compose-Spec form
  `deploy.resources.reservations.devices` with `driver: nvidia`** — the *same*
  path #10 landed on. This is **not** iX-proprietary, not CDI, not the legacy
  `runtime: nvidia` form. Verified by reading the renderer source in
  `github.com/truenas/apps` at `library/2.3.8/resources.py`
  (`Resources._auto_add_gpus_from_values`): the GPU dropdown writes GPU UUIDs
  into `device_ids` and emits exactly
  `{capabilities: [gpu], driver: nvidia, device_ids: [...]}`. The Jellyfin app
  template itself does **zero** GPU plumbing — it's all in the shared library.

- **Point 3 — system-wide runtime (high confidence, decisive):** Installing the
  NVIDIA driver via the TrueNAS web UI **does register the `nvidia` runtime with
  the system dockerd** — visible to `docker info` and usable by **any** Docker
  container, including a Dockhand Compose stack. The runtime is **not** injected
  per-app by iX's framework. Two independent lines of evidence: (a) iX support
  themselves direct users to run `midclt call -job docker.update '{"nvidia":
  true}'`, which reconfigures the **system Docker service** (the same dockerd
  Dockhand reaches via `/var/run/docker.sock`); (b) the canonical community
  guide for "classic Docker on TrueNAS" runs the standard
  `nvidia-ctk runtime configure --runtime=docker` against that same daemon and
  registers the standard named runtime in `/etc/docker/daemon.json`. Multiple
  users in the wild run **custom Compose stacks (Portainer)** that reuse the
  *identical* `deploy.resources.reservations.devices` block and get GPU access
  with no extra host setup. **Conclusion: the Dockhand path works against the
  same runtime the curated app uses. #12's `docker run --rm --gpus all ubuntu
  nvidia-smi` should succeed — *provided the host driver actually loads* (see
  the Pascal caveat, which is a separate failure layer).**

- **Pascal reversal — load-bearing caveat (high confidence, must flag):** #10
  stated "TrueNAS 25.10 ships driver 570.172.08" for the P400. That is only
  half-true and the half it omits breaks the P400. **25.10 ships 570.172.08 with
  the OPEN GPU kernel modules**, which require the GPU System Processor (GSP)
  introduced in **Turing** (RTX 20-series / GTX 16-series / Quadro RTX). The
  **Quadro P400 is Pascal (GP107) — no GSP — and the open module refuses to
  load on it.** The 25.10 version notes list "Pascal, Maxwell, and Volta
  architectures are no longer supported" as a **breaking change**. Symptom in
  the wild: `NVIDIA-SMI has failed because it couldn't communicate with the
  NVIDIA driver` and `unknown or invalid runtime name: nvidia` (because the
  driver never loaded, the runtime was never registered). **This means #12's
  empirical test may fail at the *driver* layer on the operator's actual host,
  before any Docker/runtime question is even reached.** Workarounds exist
  (community proprietary-driver installer, or stay on 24.10) — see the
  collapsible. This does **not** change any of points 1–3 (the mechanism is
  unchanged); it changes whether the operator's specific GPU is initialized at
  all on 25.10.

- **Point 4 — topology alternative (input to #13, no decision):** Curated app
  vs Dockhand stack trade off **"GPU just works + config lives in UI state"**
  against **"config in git + GPU depends on the host runtime (which 25.10 may
  not provide for the P400 anyway)"**. Enumerated cost table below. This is
  input, not a recommendation.

<details>
<summary><b>Point 1 — App platform on 25.10 is Docker (full detail)</b></summary>

Two independent confirmations:

**24.10 "Electric Eel" release notes (the pivot):**

> *Features:* "The TrueNAS Apps feature backend moves from Kubernetes to Docker
> to streamline App deployment and management."
>
> *Upgrade Notes:* "24.10 moves the applications backend from Kubernetes to
> Docker." Also: "TrueNAS does not include a default NVIDIA GPU driver and
> instead provides a simple NVIDIA driver download option in the web
> interface." And: "Users who manually installed Docker on TrueNAS 24.04 or
> earlier can experience TrueNAS Apps failure in 24.10 or later … due to
> conflicts between the manually installed and native Docker configurations."

**25.10 "Goldeye" version notes (the continuation):**

> *Software Component Versions:* Docker Engine **28.3.1**.
>
> *25.10-BETA.1 Notable Changes:* "support for configuring external container
> registry mirrors as alternative sources for Docker images."
>
> *TrueNAS Apps:* "TrueNAS 25.10 adds an option to automatically migrate
> existing applications."

The original iX announcement (TrueNAS forums, "Apps Update – 2024-06-06")
states the migration explicitly: *"we've begun the work to replace the k3s
backend with Docker / Compose … will migrate to the new Docker Compose back
end."* 25.04 was codenamed "Fangtooth"; **25.10 is "Goldeye"** (some community
sources misspell it "Goldeneye").

So: the curated Jellyfin app on 25.10 is a **Docker Compose stack**, rendered
by iX's catalog framework into a real `docker-compose.yaml` on disk at
`/mnt/.ix-apps/app_configs/<app>/versions/<ver>/templates/rendered/docker-compose.yaml`
(path confirmed by users in the field), and applied to the **system dockerd** —
the same daemon a Dockhand stack targets. There is no separate Kubernetes, no
separate containerd-for-apps, no per-app container engine.

</details>

<details>
<summary><b>Point 2 — Exact GPU mechanism in the curated app source (full detail)</b></summary>

Read directly from `github.com/truenas/apps` (the open catalog repo iX uses to
"render and update the Apps catalog"). Current library version is `2.3.8`.

**The renderer that emits the GPU block — `library/2.3.8/resources.py`,
`Resources._auto_add_gpus_from_values`:**

```python
def _auto_add_gpus_from_values(self):
    resources = self._render_instance.values.get("resources", {})
    gpus = resources.get("gpus", {}).get("nvidia_gpu_selection", {})
    if not gpus:
        return

    for pci, gpu in gpus.items():
        if gpu.get("use_gpu", False):
            if not gpu.get("uuid"):
                raise RenderError(f"Expected [uuid] to be set for GPU in slot [{pci}] ...")
            self._nvidia_ids.add(gpu["uuid"])

    if self._nvidia_ids:
        if not self._reservations:
            self._reservations["devices"] = []
        self._reservations["devices"].append(
            {
                "capabilities": ["gpu"],
                "driver": "nvidia",
                "device_ids": sorted(self._nvidia_ids),
            }
        )
```

That is the **entire** GPU plumbing. Three things to notice:

1. It is the **modern Compose-Spec** `deploy.resources.reservations.devices`
   form, character-for-character the same shape #10's compose block uses
   (`driver: nvidia`, `capabilities: [gpu]`, `device_ids: [...]`). It uses
   `device_ids` (GPU UUIDs) rather than `count` because iX lets the user pick
   specific GPUs from a dropdown populated by `midclt call app.gpu_choices`.
2. There is **no** `runtime: nvidia` key, no `NVIDIA_VISIBLE_DEVICES` env, no
   CDI spec, no iX-proprietary device injection. The `Devices` class
   (`library/2.3.8/devices.py`) handles `/dev/dri` passthrough for **Intel/AMD**
   iGPUs (`use_all_gpus` → mount `/dev/dri` and optionally `/dev/kfd` for AMD
   ROCm) — a *different* code path that is **not** used for NVIDIA.
3. The Jellyfin app template (`ix-dev/community/jellyfin/templates/docker-compose.yaml`)
   does **nothing** GPU-specific. It calls `{{ tpl.render() | tojson }}` and
   the library's `Resources` auto-adds the GPU block from the
   `resources.gpus.nvidia_gpu_selection` values the UI wrote. The questions
   schema (`ix-dev/community/jellyfin/questions.yaml`) just references the
   shared definition `$ref: ["definitions/gpu_configuration"]`.

**Implication for Dockhand:** because the curated app and a Dockhand stack both
speak the same Compose-Spec dialect to the same dockerd, **the compose block in
#10 is byte-for-byte compatible with what iX's own framework emits.** There is
no secret sauce in the curated app that Dockhand cannot reproduce. The only
thing iX adds is the *UI dropdown* that writes the UUID into `device_ids` — a
convenience, not a capability gate.

The field semantics (confirmed against the Docker Compose spec in #10) still
hold: `capabilities: [gpu]` is **required**; `count` and `device_ids` are
**mutually exclusive**; with a single GPU you can use `count: 1` instead of
`device_ids`, which is what #10's block does and what the forum user
`rambro1stbud` reports working in their custom Jellyfin compose.

</details>

<details>
<summary><b>Point 3 — System-wide runtime (THE decisive question, full detail)</b></summary>

**Verdict: system-wide.** The `nvidia` runtime is registered with the system
dockerd and is usable by any Docker container, including a Dockhand Compose
stack. It is **not** injected per-app by iX's framework. Confidence: **high**,
on two independent lines of evidence.

**Evidence A — iX support's own fix uses the system Docker service.**

In the TrueNAS forum thread *"Docker apps and UUID issue with NVIDIA GPU after
upgrade to 24.10 or 25.04"* (started by iX staff **Chris Peredun / HoneyBadger**),
user **Christopher Powell** recounts that iX support (Kryssa, on Discord) had
him run, from the host shell:

```
midclt call -job docker.update '{"nvidia": true}'
```

`midclt` is the TrueNAS middleware client. `docker.update` is the middleware
method that reconfigures the **system Docker service** — the same dockerd that
listens on `/var/run/docker.sock`, which is the socket Dockhand mounts. The
effect is to flip NVIDIA support on for the daemon. Powell reports the command
temporarily made "all my apps disappear" (a docker restart), then everything
came back and `nvidia-smi` worked. This is a **system-wide** reconfiguration,
not a per-app injection. The same command is independently cited by
`rambro1stbud` in the *"Electric Eel Nvidia gpu passthrough support"* thread as
the step that made NVIDIA passthrough "available as an option when installing
Jellyfin, and hardware transcoding is tested and working."

**Evidence B — the community "classic Docker on TrueNAS" path uses the standard toolkit against the same daemon.**

The Level1Techs guide *"TrueNAS Scale Native Docker & VM access to host"*
installs GPU support for custom Docker (i.e., not the curated-app path) by
detecting NVIDIA hardware and running the **canonical NVIDIA Container Toolkit
command**:

```bash
nvidia-ctk runtime configure --runtime=docker
```

…against the system dockerd, producing the standard named-runtime entry in
`/etc/docker/daemon.json`:

```json
"runtimes": {
  "nvidia": {
    "path": "/usr/bin/nvidia-container-runtime",
    "runtimeArgs": []
  }
}
```

This is exactly the registration `nvidia-ctk runtime configure` performs on any
Linux box (per the NVIDIA Container Toolkit install guide cited in #10). It
does **not** set `default-runtime: nvidia` — containers opt in via `--gpus` or
`deploy.resources.reservations.devices`, which the runtime hook then honors.

**Evidence C — users in the wild reuse the identical mechanism in custom stacks.**

In the *"Electric Eel Nvidia gpu passthrough support"* thread, multiple users
report running **custom Compose stacks (via Portainer, not the curated app)**
that use the *same* `deploy.resources.reservations.devices` block and get GPU
access:

- **Autopilot2301:** "I found that Jellyfin would not work with nvidia gpus …
  To get it working I had to adapt the capabilities:" then pastes
  `deploy: resources: {"reservations": {"devices": [{"capabilities":
  ["compute","utility","video"], "device_ids": ["all"], "driver":
  "nvidia"}]}}` — and notes "transcoding is now working with nvidia gpus."
- **rambro1stbud:** posts a working custom Jellyfin compose with
  `driver: nvidia`, `count: 1`, `capabilities: [gpu]`.
- **rodneysing:** runs a custom Ollama stack with both the modern `deploy`
  block and the legacy `runtime: nvidia` + `NVIDIA_VISIBLE_DEVICES` form.

The fact that *custom* Compose containers — talking to the raw docker socket,
with no participation from iX's app framework — successfully reach the GPU is
dispositive: the runtime is **system-wide**, not per-app.

**What this means for #12 / Dockhand:**

- #12's empirical probe `docker run --rm --gpus all ubuntu nvidia-smi` should
  succeed on the operator's host **if the host driver has loaded** (see the
  Pascal caveat — that's a different failure layer).
- The compose block in #10 (`deploy.resources.reservations.devices`,
  `driver: nvidia`, `count: 1`, `capabilities: [gpu]`) is **identical in shape
  to what iX's own catalog emits**. A Dockhand stack using it will hit the
  same runtime, the same daemon, the same code path as the curated Jellyfin
  app.
- **No extra toolkit install is needed on the host** for the Dockhand path —
  iX's own driver+runtime registration already covers it. This **de-risks and
  partially reverses** #10's "single biggest unknown" (which assumed the
  toolkit was missing and `apt` was the only install path). The toolkit/runtime
  is provisioned by iX's middleware when the NVIDIA driver is enabled; the
  `apt`-disabled caveat in #10 is about *manual* installs, not about the
  iX-managed runtime.

**Caveat on confidence (what only the live host can confirm):**

- Whether the operator's specific host currently has the runtime registered
  depends on whether the driver was *successfully* enabled via the UI. The
  Pascal/open-module issue (below) can prevent the driver from loading, which
  in turn prevents the runtime from registering — manifesting as
  `unknown or invalid runtime name: nvidia`. So the runtime is *capable* of
  being system-wide, but its *current* presence on this host must be confirmed
  by `docker info | grep -i runtime` (or the #12 probe). Handed to #12.

</details>

<details>
<summary><b>The Pascal reversal — 25.10 open modules drop the P400 (full detail)</b></summary>

This is **not** one of the ticket's four points, but it materially affects
whether points 2–3 produce a working GPU on the operator's actual hardware. #10
asserted "TrueNAS 25.10 ships driver 570.172.08" supporting the P400, citing a
forum thread. That is true of the *driver package version* and **false** about
P400 support. The correction:

**25.10 version notes ("25.10.0 Notable Changes"):**

> *"TrueNAS 25.10 switches to open GPU kernel drivers,"* supporting
> *"Turing and newer (RTX/GTX 16-series+)."* Under **NVIDIA GPU Support**:
> *"compatible with Turing architecture and later GPUs … GPUs based on earlier
> architectures, including Pascal … are not supported by the NVIDIA open
> drivers."* Listed as a **breaking change**: *"Pascal, Maxwell, and Volta
> architectures are no longer supported."*

**Why:** NVIDIA's open GPU kernel modules depend on the GPU System Processor
(GSP), first introduced in Turing. Pre-Turing GPUs (Pascal GP1xx, Maxwell
GM1xx, Volta) have no GSP, so the open `nvidia.ko` refuses to load on them.
NVIDIA's own open-modules README states this directly: *"The open kernel
modules cannot support GPUs before Turing, because the open kernel modules
depend on the GPU System Processor (GSP) first introduced in Turing."*

**Symptom in the wild (TrueNAS forum, "25.10.2.1 Goldeye Docker NVIDIA driver
issues"):** user `gswhiteuk` with a Quadro P620 (also Pascal) hits
`Error response from daemon: unknown or invalid runtime name: nvidia` —
**because the driver never loaded, the runtime was never registered.** This is
the *exact* failure mode that would make #12's probe fail on the operator's
P400, and it has **nothing to do** with the curated-vs-Dockhand question.

**The same driver version (570.172.08) DOES support Pascal via the proprietary
modules** — NVIDIA ships both. The breakage is specifically iX's choice to
switch TrueNAS 25.10 to the **open** modules. So the operator's #10-derived
belief ("570.172.08 is present, floor cleared") is a half-truth: the package
is present, the open module inside it does not bind to the P400.

**Workarounds (community-sourced; not vouched for, listed for completeness):**

1. **Community proprietary-driver installer** — `zzzhouuu/truenas-nvidia-drivers`
   (GitHub), install page `truenas-drivers.zhouyou.info`. A one-line
   `curl … | bash` that detects the TrueNAS version and installs the matching
   **proprietary** driver so Pascal/Maxwell cards work on 25.10. **Caveat:**
   must be re-run after each TrueNAS system update (updates overwrite the
   module); Secure Boot must be off. Multiple forum threads
   ("Pascal GPU on Version 25.10.3.1", "Nvidia Driver not installing",
   "25.10 broke nvidia driver support for Quadro M2000",
   "TrueNAS Scale 25.10.0.1 and Nvidia Quadro P620") point here.
2. **Disable the in-app driver download** (Apps → Configuration → Settings →
   uncheck "Download NVIDIA drivers"), reboot, then install the proprietary
   driver manually.
3. **Stay on / roll back to 24.10 "Electric Eel"** (which shipped the
   proprietary driver and supports Pascal). Roll back via System → Boot →
   select a previous boot environment. Temporary; 24.10 is being deprecated.
4. **Replace the GPU** with a Turing-or-newer card (e.g. GTX 1650, T400, RTX
   A400, Quadro RTX-series). This is the "permanent" fix iX staff point to.

iX has so far **not accepted** the feature request to add a UI toggle for
open-vs-proprietary driver selection.

**Impact on this ticket's answers:** none of points 1–3 change. The mechanism
(Docker + `deploy.resources.reservations.devices` + system-wide `nvidia`
runtime) is identical whether the GPU is a P400 or an RTX 4000. What changes is
the **prerequisite**: on 25.10 + P400, the operator must first solve the
driver-loading problem (workarounds above) before *any* path — curated app or
Dockhand — can use the GPU. This is a **#10/#12** concern, not a #15 concern,
but it would be malpractice to write this doc without flagging it.

</details>

<details>
<summary><b>Point 4 — Curated app vs Dockhand stack (input to #13, full detail)</b></summary>

This is **input** to the topology grilling in #13. No decision is made here.

| Dimension | Curated TrueNAS app | Dockhand `truenas/apps/jellyfin` stack |
|---|---|---|
| **Config location** | TrueNAS UI state (middleware DB / `/mnt/.ix-apps`). **Not in git.** Breaks the repo's gitops convention. | `truenas/apps/jellyfin/compose.yaml` in this repo. In git, reviewed, diffable. |
| **GPU setup** | Dropdown picks the GPU; iX's renderer emits the `deploy.resources.reservations.devices` block. Works out of the box — *if the host driver loads* (Pascal caveat applies equally). | Manual `deploy.resources.reservations.devices` block (per #10). Identical mechanism; identical host prerequisite. |
| **Image / version control** | Pinned by iX's catalog train; updates gated on iX publishing a new app version. Limited pinning. | Full control: pin `lscr.io/linuxserver/jellyfin:<tag>` exactly. Updates on your schedule. |
| **Mounts / volumes** | ix-volumes or host-path via UI fields. Schema constrained by the app's `questions.yaml`. | Arbitrary; matches the repo's per-stack conventions (`networks: [proxy]`, `.env.example`, etc.). |
| **Secrets** | TrueNAS app credential fields. | Dockhand encrypted secrets (per the `dockhand-capabilities` briefing), with the daemon-restart caveat. |
| **Edge proxy integration** | None first-class; would need a separate Caddy label hack or a manual network join to `proxy`. | Native: follows the repo convention (Caddy labels, `proxy` external network). |
| **Lifecycle / drift** | Managed by iX's app framework; manual edits to the rendered compose are overwritten on next app update. | Dockhand diff-gated deploys; manual Deploy button; git is source of truth. |
| **Support** | iX-supported (stable train) / community-supported (community train). Jellyfin is in the **community** train. | Forum-only (Dockhand itself is a community app). |
| **GPU prerequisite on 25.10 + P400** | Identical: the open-module breakage hits both. Curated-app convenience does not bypass the driver layer. | Identical. |

**The honest read for #13:** the curated app's only real advantage is "GPU
dropdown just works" — and per point 3, **that advantage evaporates** because
the Dockhand path uses the *same* runtime the curated app does, with no extra
host setup. Once that's established, the trade collapses to the repo's existing
gitops convention vs config-in-UI-state, and the repo has already decided (per
`truenas/apps/README.md` and decision #5) that stacks live in git under
Dockhand. The curated app would be a **regression** on that convention unless
#13 explicitly re-opens it.

The one scenario where the curated app wins decisively: **if the operator
cannot or will not solve the Pascal-driver problem themselves**, iX's driver
management (and the community proprietary-driver installer, which targets the
same iX-managed driver slot) is at least a more integrated path than a Dockhand
stack that depends on a host runtime the operator had to cobble together. But
that's a #10/#12 problem, not a topology problem.

</details>

## Confidence notes

- **Doc-backed, highest confidence:** Point 1 (24.10 + 25.10 release notes —
  Docker platform, Docker Engine 28.3.1, codenames Electric Eel / Fangtooth /
  Goldeye). Point 2 (read the renderer source `library/2.3.8/resources.py` in
  `github.com/truenas/apps`; the GPU block is emitted verbatim as modern
  Compose-Spec). Pascal reversal (25.10 version notes state the breaking
  change explicitly; NVIDIA's open-modules README states the GSP/Turing
  requirement; multiple forum threads confirm the P620/P2000 symptom).
- **High confidence, multi-source:** Point 3 (iX staff-initiated
  `midclt docker.update {"nvidia": true}` reconfigures the system dockerd;
  community "classic Docker" guide uses standard `nvidia-ctk runtime configure`
  against the same daemon; multiple users run custom Compose stacks that reuse
  the identical mechanism). The runtime is system-wide, not per-app.
- **Cannot confirm from docs alone (→ #12):** whether the operator's *current*
  host actually has (a) a successfully loaded driver for the P400 on 25.10's
  open modules (likely **no**, per the Pascal reversal), and (b) the `nvidia`
  runtime registered in `docker info`. Both are live-host checks; the runtime
  being system-wide-capable does not guarantee it is currently present if the
  driver failed to load.
- **Community-sourced, lower confidence:** the specific workarounds for the
  Pascal problem (`zzzhouuu/truenas-nvidia-drivers` installer; the
  "uncheck Download NVIDIA drivers" dance). Widely cited on the forums but not
  iX-endorsed; listed for completeness, not recommended.

## Sources

- [TrueNAS SCALE 24.10 (Electric Eel) Release Notes](https://www.truenas.com/docs/scale/24.10/gettingstarted/scalereleasenotes/) — Docker pivot, NVIDIA driver via UI
- [TrueNAS SCALE 25.10 (Goldeye) Version Notes](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalereleasenotes/) — Docker Engine 28.3.1, open GPU kernel modules breaking change
- [TrueNAS Apps Catalog source — `github.com/truenas/apps`](https://github.com/truenas/apps) — `library/2.3.8/resources.py` (`_auto_add_gpus_from_values`), `library/2.3.8/devices.py`, `ix-dev/community/jellyfin/questions.yaml`, `ix-dev/community/jellyfin/templates/docker-compose.yaml`
- [TrueNAS forum — Apps Update 2024-06-06 (k3s → Docker announcement)](https://forums.truenas.com/t/apps-update-2024-06-06/6041)
- [TrueNAS forum — Electric Eel NVIDIA GPU passthrough support](https://forums.truenas.com/t/electric-eel-nvidia-gpu-passthrough-support/11797) — `midclt docker.update {"nvidia": true}`, custom-Compose GPU reuse (Autopilot2301, rambro1stbud, rodneysing)
- [TrueNAS forum — Docker apps and UUID issue with NVIDIA GPU after upgrade to 24.10/25.04](https://forums.truenas.com/t/docker-apps-and-uuid-issue-with-nvidia-gpu-after-upgrade-to-24-10-or-25-04/22547) — iX staff (HoneyBadger/Chris Peredun), `midclt docker.update` from iX support
- [TrueNAS forum — 25.10.2.1 Goldeye Docker NVIDIA driver issues](https://forums.truenas.com/t/25-10-2-1-goldeye-docker-nvidia-driver-issues/64220) — open modules + Pascal breakage, `unknown or invalid runtime name: nvidia`
- [TrueNAS forum — Pascal GPU on Version 25.10.3.1](https://forums.truenas.com/t/pascal-gpu-on-version-25-10-3-1/66392)
- [TrueNAS forum — NVIDIA compatible driver test for TrueNAS 25.10 Goldeye](https://forums.truenas.com/t/nvidia-compatible-driver-test-for-truenas-25-10-goldeye/53395) — community proprietary-driver workaround thread
- [zzzhouuu/truenas-nvidia-drivers (community legacy driver installer)](https://github.com/zzzhouuu/truenas-nvidia-drivers) · [truenas-drivers.zhouyou.info](https://truenas-drivers.zhouyou.info)
- [Level1Techs — TrueNAS Scale Native Docker & VM access to host (guide)](https://forum.level1techs.com/t/truenas-scale-native-docker-vm-access-to-host-guide/190882) — `nvidia-ctk runtime configure --runtime=docker` against system dockerd, daemon.json named-runtime entry
- [NVIDIA — Open Linux Kernel Modules README (GSP/Turing requirement)](https://download.nvidia.com/XFree86/Linux-x86_64/515.43.04/README/kernel_open.html) · [NVIDIA open-gpu-kernel-modules](https://github.com/nvidia/open-gpu-kernel-modules)
- [NVIDIA Container Toolkit — install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) — `nvidia-ctk runtime configure` semantics
- [Docker — Run Compose services with GPU access](https://docs.docker.com/compose/how-tos/gpu-support/) — `deploy.resources.reservations.devices` spec
- [Repo cross-reference — `research/jellyfin-gpu-passthrough.md` (#10 findings)](./jellyfin-gpu-passthrough.md) — the Pascal reversal above amends its host-availability claim
- [Repo cross-reference — `research/dockhand-capabilities.md`](./dockhand-capabilities.md)
