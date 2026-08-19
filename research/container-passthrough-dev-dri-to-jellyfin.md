# Container passthrough of /dev/dri/renderD128 to Jellyfin under Dockhand

> Resolves wayfinder ticket #4. Compiled 2026-08-11 via `/research` probe.
> Scope: the Dockhand-on-TrueNAS-SCALE deployment model documented in [dockhand-capabilities.md](./dockhand-capabilities.md) (Dockhand drives plain `docker compose`; one `compose.yaml` per stack under `truenas/apps/<stack>/`). Target hardware: Intel Arc A380. Target codec path: QSV via `jellyfin-ffmpeg`.

## TL;DR

For the Arc A380 + Jellyfin under Dockhand, use the **official `jellyfin/jellyfin` image** (not linuxserver) and this block in `truenas/apps/jellyfin/compose.yaml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    # image runs as uid 410542:410542 by default — leave user: unset
    group_add:
      - '107'                                  # host `render` GID — confirm on the box first
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0          # only if you want OpenCL/tonemapping
    # security_opt: not required under Docker Engine on SCALE
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/media:/media:ro
    network_mode: host                          # or publish 8096 explicitly
    restart: unless-stopped
```

Detection step the operator runs first, on the TrueNAS shell:

```sh
stat -c '%g' /dev/dri/renderD128            # prints the render GID — use this verbatim
getent group render | cut -d: -f3           # cross-check (often 107 on Debian family)
ls -l /dev/dri                              # confirm card0 + renderD128 exist and owners
```

Then verification inside the running container:

```sh
docker compose exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo
docker compose exec jellyfin /usr/lib/jellyfin-ffmpeg/ffmpeg \
  -v verbose -init_hw_device vaapi=va -init_hw_device opencl@va
# trigger a transcode in the UI, then on the host:
sudo intel_gpu_top                          # Video/0 + Video/1 engines should spike
```

Key decisions, in one line each:

- **Device binding:** minimum sufficient is `renderD128` alone (pure QSV). Bind `card0` too if you want HDR/HLG tonemapping via OpenCL. Do **not** bind the whole `/dev/dri` directory unless you have a reason — explicit is better.
- **GID:** use `group_add: ['<render GID>']` with the host's numeric GID hardcoded after running `stat`. On SCALE that is usually **107**, not 104. Varies; always check.
- **User:** leave the image's default `410542:410542` alone. Do **not** run as root. `group_add` is the supported way to grant render access to that non-root user.
- **security_opt:** not needed under Docker Engine on SCALE. No AppArmor/seccomp profile ships by default that blocks DRI. Skip it.
- **Dockhand:** it passes `devices`, `group_add`, and `security_opt` through to `docker compose` unchanged. It is a thin compose driver; no field sanitization observed.

## Device passthrough mechanics at the compose level

Three valid approaches, in increasing scope:

1. **`devices: - /dev/dri/renderD128:/dev/dri/renderD128`** — minimum sufficient for QSV encode/decode. This is what the Jellyfin docs use as their canonical Intel example, and what the `jellywatch.app` 2026 guide pins as the working minimum ("Map renderD128 specifically, not just card0"). Choose this if you only want H264/HEVC/AV1/VP9 QSV transcode.
2. **Add `devices: - /dev/dri/card0:/dev/dri/card0`** — needed if you also want **OpenCL-based HDR/Dolby Vision tonemapping**. The official Jellyfin docs' commented SCALE snippet shows both lines. The Arc A380 supports 3D-LUT tonemapping, so this is worth binding if you transcode HDR→SDR.
3. **`devices: - /dev/dri:/dev/dri`** — bind the whole directory. Works, less explicit. The LSIO image docs and many forum posts use this; fine, but on a multi-GPU host you may inadvertently expose an unrelated adapter.

`device_cgroup_rules` (`c 226:0 rmw`, `c 226:128 rmw`) is the **legacy escape hatch** for environments where `devices:` doesn't grant cgroup access — Synology DSM, some Swarm setups, older Docker. On modern Docker Engine (which is what SCALE runs) `devices:` already programs the cgroup rule, so `device_cgroup_rules` is redundant. Skip it unless you hit a specific limitation.

**card0 vs renderD128 — what Jellyfin actually uses:**

- **renderD128** is the render node. QSV and VA-API encode/decode go through it. Pure transcoding needs nothing else.
- **card0** is the KMS/display node. It is required by **OpenCL** (Intel Compute Runtime / NEO) which Jellyfin's HDR/DV tonemapping path uses. If you enable "Hardware tonemapping" in the Jellyfin dashboard without card0 bound (and the OpenCL runtime present), it will silently fall back or fail.
- One community workaround for Arc on Unraid: select `card0` as the VA-API device in the dashboard rather than `renderD128`. Treat that as a debug fallback, not the primary config.
- Kernel gotcha (Jellyfin known-issues page): kernels 5.18 through 6.1.3 have an i915 driver bug that locks up the GPU under OpenCL tonemapping. SCALE's kernel is well past that range today; flag only if the host ever regresses.

**Recommendation for ticket 07:** bind both `card0` and `renderD128` explicitly. It costs nothing and unlocks tonemapping.

## Group membership / GID

SCALE's `/dev/dri/renderD128` is owned `root:render` and mode `crw-rw----` (226,128). The numeric GID of `render` is **not fixed across distros**:

- Debian/Ubuntu family: commonly **107** (the ticket's "104" guess is wrong for SCALE — verify; it's 107 on Dragonfish/Electric Eel). TrueNAS forum threads consistently report `render:x:107:`.
- Some Ubuntu server installs: **122** (this is the value the Jellyfin docs use in their generic Intel example).
- RHEL/Fedora family: **107** by convention but often **video** (GID 39) instead.
- Arch: often **989** or higher.
- 104 is sometimes seen on older Debian systems for `render`, but is not what SCALE currently ships.

**Detection is non-negotiable.** Run on the host before writing the GID into compose:

```sh
stat -c '%g' /dev/dri/renderD128     # authoritative — the GID that owns the device file
getent group render | cut -d: -f3    # name → GID cross-check
ls -l /dev/dri                       # sanity: confirm the group name is `render`
```

Use the `stat` output. It is the single source of truth and survives cases where the device is owned by a group other than `render` (some hosts use `video`).

### Three options for the container user

| Option | How | Verdict |
|---|---|---|
| **`group_add: ['<GID>']`** with default non-root user | Keep image default (410542:410542), add render GID via `group_add` | **Recommended.** This is the pattern in the Jellyfin docs. Least privilege, no PUID dance, no root. |
| Run as root (`user: 0:0` or no user directive on a root image) | Drop `group_add`, Jellyfin just works | Works but security smell, and the official image isn't root anyway so this would be an override. Skip. |
| linuxserver image + `PUID`/`PGID` env | `image: linuxserver/jellyfin`, `environment: PUID=...; PGID=...` | LSIO auto-grants the `abc` user device access, sidestepping `group_add`. Viable but it's a different image, different volume paths (`/config` semantics differ), and loses the bundled `jellyfin-ffmpeg` + OpenCL runtime that the official image ships. Not worth switching for this. |

**Recommendation:** keep the official `jellyfin/jellyfin` image at its default `410542:410542` user, and add the host render GID via `group_add: ['107']` (or whatever `stat` returned). The Jellyfin docs' TrueNAS section is explicit: *"Unless you set the container to run as root, you need to add the render group ID to the container with group_add."*

### The official image's user, demystified

The `jellyfin/jellyfin` image runs as **UID/GID 410542** (user `jellyfin`) by default. The high UID is deliberate: it avoids collisions with normal host users. Do **not** override `user:` unless you have a permissions reason (e.g. matching a media tree owned by another UID). If you do override it, your config/cache/media volumes must be writable by the UID you pick. `group_add` is orthogonal to `user:` — it adds supplementary groups to whatever user the container runs as.

## Dockhand compose feature coverage

Dockhand is a thin UI over `docker compose`. The capability briefing ([dockhand-capabilities.md](./dockhand-capabilities.md)) establishes the model: each stack is one compose directory, Dockhand shells out to `docker compose` on deploy. There is **no value-level sanitization** of compose fields in the documented surface — Dockhand does not rewrite `devices`, `group_add`, `security_opt`, `cap_add`, or any of the standard service-level keys.

What Dockhand **does** touch:

- Injects marked secrets as shell env vars before invoking compose.
- Diffs the stack directory to decide whether to redeploy.
- Surfaces metadata (Traefik labels, network graph).

What Dockhand **does not** do:

- No field stripping on `devices:`/`group_add:`/`security_opt:`.
- No mandatory schema validation that would reject these keys.
- No proxying of the device through a different mechanism — it's the literal compose you wrote, applied with `docker compose`.

Confidence here is high but not absolute: the public manual does not enumerate a "supported compose keys" matrix, so this is inferred from (a) the stated architecture ("drives plain docker compose"), (b) absence of any documented sanitizer, (c) community reports of GPU stacks running under Dockhand without special config. If a future Dockhand release adds schema validation, this would be the place it breaks.

## AppArmor / security_opt

Docker Engine on Linux applies a default **seccomp** profile and, where AppArmor is available, a default AppArmor profile. Neither blocks `/dev/dri` access **when the device is explicitly bound via `devices:`**. The default Docker AppArmor profile is `docker-default`, which is a denylist that permits device access granted through `--device`. The seccomp profile permits the ioctls VA-API/QSV needs.

- **SCALE specifically:** SCALE does not ship a custom AppArmor profile that restricts DRI, and TrueNAS does not confine Docker apps behind an AppArmor harness. Forum reports of GPU passthrough working under standard SCALE Docker use only `devices:` + `group_add:`, no `security_opt`.
- The Jellyfin docs' Docker Intel example includes **no `security_opt`**. The LSIO image docs include **no `security_opt`**. The `jellywatch.app` 2026 guide includes **no `security_opt`** for Intel; it mentions `--privileged` only as a last-resort debug step, which we do not want.
- **Kubernetes is different:** the Jellyfin docs note that under k8s the container must run privileged. That's a k8s/devices.k8s.io constraint and does not apply to us.

**Recommendation:** omit `security_opt`. If (and only if) you hit a weird denial in a future Dockhand/SCALE combo, the escape hatch is `security_opt: ['apparmor:unconfined']` — but you almost certainly won't need it.

## The TrueNAS first-party app alternative (out of scope, for contrast)

TrueNAS ships a first-party **Jellyfin** app in the ix-systems official catalog. It exposes a "Resource Configuration" section with a checkbox: **"Passthrough available (non-NVIDIA) GPUs"**. Ticking it makes the app framework inject `/dev/dri` into the container and add the appropriate `render`/`video` group. There's also an NVIDIA path that requires installing NVIDIA drivers via `Configure > Settings > Install NVIDIA Drivers`.

Why we're not using it: the constraint on this effort is that **git is the source of truth via Dockhand**. The ix-app catalog model stores app config in the SCALE middleware DB (and in ix-app YAML for Kubernetes-backed apps in newer SCALE), not in our git repo. Diff-gated redeploys, branch promotion, and reviewability all favor our own compose under Dockhand. The first-party app is fine for someone who wants point-and-click; it does not fit our gitops-shaped hole. The hardware passthrough mechanics are the same — `devices:` + `group_add:` under the hood — the catalog just hides them behind a toggle.

## Verification inside the container

After `dockhand deploy` brings the stack up:

1. **Device presence:**

   ```sh
   docker compose exec jellyfin ls -l /dev/dri
   # Expect card0 and renderD128 with the GID you set in group_add.
   ```

2. **VA-API / QSV codec surface** (this is the load-bearing check). The official image bundles `jellyfin-ffmpeg` with its own libva and Intel media drivers; no need to install `intel-media-va-driver-non-free` separately:

   ```sh
   docker compose exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo
   ```

   Look for the encoder/decoder entries. On Arc A380 (Alchemist, `i915` + `intel-media-va-driver` for DG2/Xe-HPG), expect:
   - `VAProfileH264Main / High` — enc + dec
   - `VAProfileHEVCMain / Main10` — enc + dec
   - `VAProfileVP9Profile0/1/2/3` — dec
   - `VAProfileAV1Profile0` — enc + dec (A380 has AV1 enc/dec; this is its party trick)
   If you see `libva info: va_openDriver() returns -1` or `failed to load driver for i915`, the GID is wrong or the device isn't bound.

3. **OpenCL for tonemapping** (only if you bound card0):

   ```sh
   docker compose exec jellyfin /usr/lib/jellyfin-ffmpeg/ffmpeg \
     -v verbose -init_hw_device vaapi=va -init_hw_device opencl@va
   ```

   Expect a line showing the OpenCL platform (Intel NEO) and a successful `va` device init. If OpenCL init fails but VA-API works, switch the dashboard tonemapping mode from OpenCL to VPP.

4. **End-to-end smoke test.** In Jellyfin dashboard → Playback → Transcoding, set "Hardware acceleration" to **Intel QuickSync (QSV)**, enable the codecs your hardware lists, save. Play a file that forces a transcode (force a bitrate limit on the client). On the host:

   ```sh
   sudo intel_gpu_top
   ```

   The `Video/0` and `Video/1` engines should light up. The Jellyfin session row should show an `HW` badge.

## Deliverable for ticket 07

The exact compose fragment for `truenas/apps/jellyfin/compose.yaml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    # user: intentionally unset — image default is uid/gid 410542 (non-root)
    group_add:
      - '107'                                       # <- replace with `stat -c '%g' /dev/dri/renderD128` output
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128     # QSV encode/decode (required)
      - /dev/dri/card0:/dev/dri/card0               # OpenCL tonemapping (recommended for Arc A380)
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/media:/media:ro
    network_mode: host
    restart: unless-stopped
    # security_opt: none required under Docker Engine on SCALE
```

Plus the operator-run prereq step (document in the ticket body):

```sh
stat -c '%g' /dev/dri/renderD128      # paste output as the group_add value above
```

And the post-deploy verification block from the section above.

## Confidence notes

- **Doc-backed (high confidence):** minimum-sufficient `renderD128` binding, `group_add` pattern with host GID, default non-root UID 410542 in the official image, `security_opt` not required under Docker, SCALE uses GID 107 for `render` on current releases, first-party app's "Passthrough available GPUs" toggle, `vainfo` / `jellyfin-ffmpeg` verification commands, kernel 5.18–6.1.3 OpenCL lockup caveat.
- **Inferred from architecture (high but not absolute):** Dockhand passes `devices`/`group_add`/`security_opt` through unchanged. No field sanitization documented; no community report of one. If a future Dockhand release adds schema validation, this is the place it breaks.
- **Empirical, your-mileage-may-vary:** the render GID is **107 on SCALE today** but the only authoritative answer is `stat` on the actual box. Do not trust any doc (including this one) over that number.

## Sources

[Jellyfin docs — HWA on Intel GPU](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/) · [Jellyfin docs — TrueNAS SCALE installation](https://jellyfin.org/docs/general/installation/advanced/truenas/) · [Jellyfin docs — Container installation](https://jellyfin.org/docs/general/installation/container/) · [Jellyfin docs — Hardware acceleration overview](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/) · [Jellyfin docs — Known hardware acceleration issues](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/known-issues/) · [LSIO docker-jellyfin docs](https://docs.linuxserver.io/images/docker-jellyfin/) · [jellywatch.app — Jellyfin Docker GPU passthrough 2026](https://jellywatch.app/blog/jellyfin-docker-gpu-passthrough-intel-nvidia-amd-2026) · [TrueNAS forums — Plex HW transcoding 25.04.1](https://forums.truenas.com/t/plex-hardware-transcoding-in-25-04-1-container-instance/48912) · [TrueNAS forums — Jellyfin CE HW encoding setup](https://forums.truenas.com/t/radeon-gpu-jellyfin-truenas-community-edition-hardware-encoding-setup/61571) · [LSIO discourse — Jellyfin Intel HWA](https://discourse.linuxserver.io/t/help-with-jellyfin-intel-hardware-acceleration/8152) · [r/unRAID — Intel Arc HW transcoding](https://www.reddit.com/r/unRAID/comments/1eptyfm/) · [ryanbritton.com — Synology Docker HW transcode without privileged](https://ryanbritton.com/2022/05/hardware-transcoding-on-synology-docker-without-privileged-mode/) · [dockhand-capabilities.md](./dockhand-capabilities.md)
