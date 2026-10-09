# Strata — deployment runbook

Two inference profiles of **Qwen3.8-Flash-Next** on the AMD Radeon AI PRO R9700
(RDNA4, gfx1201), served by [Strata](https://github.com/Niko1221/Strata) over
its HIP backend. This runbook covers **both** stacks.

| Stack | Model | Context | KV | Vision | RAM cap | Auto-start |
|---|---|---:|---|---|---:|---|
| `strata-daily` | Swift 1.5 IQ2_XS | 64K | int8 | cpu | 64 GB | **yes** |
| `strata-coding` | Coder IQ1_M | 128K | k8v4 | cpu | 48 GB | no |

## Why two stacks, not one stack with two services

Verified against Docker before committing to this shape:

| Service config | Started by `docker compose up -d`? | Container created? |
|---|---|---|
| `restart: "no"` | **yes** | yes |
| `deploy.replicas: 0` | no | **no** — Dockhand could not start it later |
| `profiles: [coding]` | no | **no** — profile-gated |

`restart: "no"` only governs *restarts*; it does not stop an initial `up`. The
other two mechanisms leave no container to start by name, so Dockhand's
per-container start button would have nothing to act on. A single stack
therefore starts **both** profiles on deploy, and two ROCm processes with no
VRAM isolation surface as a hang or a stuck 100%-GPU state rather than a clean
OOM. Separate stacks give Dockhand independent start/stop, which is the switch
you actually want.

**`strata-daily`, `strata-coding` and `comfyui` are mutually exclusive.** Stop
one in Dockhand before starting another. All three publish a host port, so a
mistaken second start also fails fast on the bind.

## Why Strata rather than llama.cpp

`research/llm-engine-rdna4-options.md` and `research/llm-model-shortlist-32gb.md`
both assumed weights must be VRAM-resident, so both recommended llama.cpp and
both **excluded** Flash-Next ("weights alone are ~47 GB at IQ3"). Strata's
RAM/SSD expert offload invalidates that premise, which only became viable here
with 128 GB of RAM and an Optane. The llama.cpp research stands as the fallback
path if Strata disappoints.

## 1. Datasets

Both run on the TrueNAS host. Nothing needs a chown: the Strata image runs as
root, as the ComfyUI image does.

### Fast pool — the model shards

```sh
zpool create -o ashift=12 cistern /dev/disk/by-id/<optane-900p-id>
zfs create -p -o recordsize=1m -o atime=off cistern/apps/strata/models
```

Single disk, no redundancy, on purpose: everything under it is re-downloadable
from Hugging Face at a pinned revision. `recordsize=1m` suits the large
sequential shard reads; `atime=off` avoids write amplification on a read-mostly
dataset.

The 900P is the right drive for this: it shares the 905P's controller
generation and its sub-10 µs read latency, differing mainly in rated
throughput (2500 vs 2600 MB/s sequential read, 550k vs 575k random-read IOPS)
and capacity. At queue depth 1 — which is where a per-token lookup-table read
lives — they are the same class, and it is ~7x faster at QD1 than any
NAND-based SSD, with none of the garbage-collection tail spikes.

`ashift=12` is deliberate even though the 900P presents 512-byte sectors and ZFS
would auto-detect `ashift=9`. Forcing 4K costs nothing here: the classic 4K
alignment penalty is a NAND read-modify-write artifact, and 3D XPoint
overwrites in place, so there is no write amplification to avoid. It is forced
for consistency with the other pools and with `recordsize=1m`.

This pool exists for exactly one reason: **shard 2**. The
`-00002-of-00002.gguf` file is a 28.8 GB n-gram/PLE lookup table that Strata
reads from disk *during inference* rather than loading into RAM. Everything
else in `/data` is read once at startup and lives in RAM afterwards — which is
why the packs and the MTP layer stay on the spinning pool.

### Existing pool — everything else

```sh
zfs create -p puddle/apps/strata
mkdir -p /mnt/puddle/apps/strata/config
```

`packs/` and `mtp/` are **derived** artefacts — `tools/iq_pack.py` rebuilds
them from the shards — so they belong on the cheap pool, not the expensive one.

### Budget

| Location | Contents | Size |
|---|---|---:|
| `cistern` | model shards + mmproj | ~119 GiB of ~250 GiB usable (~48%) |
| `puddle` | packs (~54 GiB) + mtp (~5 GiB) + config | ~59 GiB |

A 280 GB Optane is 260.8 GiB raw and about **250 GiB usable** after ZFS slop
space and metadata, so the ~119 GiB of model files sits at **~48% occupancy**.
For context, OpenZFS's actual hard guidance is only to keep free space above
10% — below 4% free the metaslabs flip from first-fit to best-fit allocation
and IOPS collapse. The 80% figure is community convention, not a ZFS
requirement. Either way there is room for roughly 80 GiB more, i.e. a second
model if you ever add one.

## 2. Pre-seed the models

Do this **before** the first Deploy, to avoid a ~128 GB download inside a
container. `--gguf-dir` is not needed: shards hand-placed with their
**original names** in the expected subdirectory are validated against their own
tensor directory and marked whole, so setup skips the download entirely.

The revision pins matter. `setup.py` reads the Hub at pinned commits, and
Swift's `main` has already moved past its pin — it has gained an IQ3_S the pin
does not have. Downloading from `main` risks a layout setup will not recognise.

```sh
MODELS=/mnt/cistern/apps/strata/models
SWIFT_REV=b22d729eae29b5796f76fb70f91aef549b9fc52c
CODER_REV=5348543e0147355ac9cbcb031184a3546350988e

mkdir -p "$MODELS/swift-IQ2_XS" "$MODELS/coder-IQ1_M"

# Swift 1.5 IQ2_XS - 68.15 GB across two shards, plus its OWN vision encoder
# (a different file from the base model's).
curl -fL --retry 3 -o "$MODELS/swift-IQ2_XS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf"
curl -fL --retry 3 -o "$MODELS/swift-IQ2_XS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf"
curl -fL --retry 3 -o "$MODELS/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf"

# Coder IQ1_M - 58.41 GB across two shards, plus the BASE mmproj it shares.
# The Coder's shard 2 is byte-identical to the base model's, so this file also
# serves any base size added later.
curl -fL --retry 3 -o "$MODELS/coder-IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf" \
  "https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF/resolve/$CODER_REV/IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf"
curl -fL --retry 3 -o "$MODELS/coder-IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf" \
  "https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF/resolve/$CODER_REV/IQ1_M/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf"
curl -fL --retry 3 -o "$MODELS/mmproj-Qwen3.8-Flash-Next-BF16.gguf" \
  "https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-Coder-GGUF/resolve/$CODER_REV/mmproj-Qwen3.8-Flash-Next-BF16.gguf"
```

Neither repo is gated, so no Hugging Face token is required. Expected layout:

```
/mnt/cistern/apps/strata/models/
├── coder-IQ1_M/
│   ├── Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf   29.6 GB
│   └── Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf   28.8 GB
├── mmproj-Qwen3.8-Flash-Next-BF16.gguf                          0.9 GB
├── mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf                    0.9 GB
└── swift-IQ2_XS/
    ├── Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf  39.8 GB
    └── Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf  28.4 GB
```

**Not pre-seedable:** the MTP draft layer (~5.2 GB, pulled by HTTP Range
requests from `Qwen/Qwen3.8-Flash-Next`'s 360 GB BF16 checkpoint) and the
per-model packs (~58 GB combined, built from the shards by `tools/iq_pack.py`).
Both run automatically on first start and cost CPU minutes, not a large
download. Setup has no `--download-only` flag.

## 3. Deploy

1. **Add the two hostnames** (manual, per `apps/README.md`): in TrueNAS
   **Network → DHCP and DNS → Hostnames**, add `strata-daily.absurdlab.dev` and
   `strata-coding.absurdlab.dev` pointing at the Caddy box LAN IP, then refresh
   Passwall's DNS with the SIGHUP snippet in `apps/README.md`.
2. **Deploy `strata-daily`** in Dockhand (stack name `strata-daily`, compose
   path `truenas/apps/strata-daily/compose.yaml`), with every key from its
   `.env.example` in the env-var panel and **every key-icon toggle OFF**.
   Dockhand builds the image (a HIP engine compile, 10–20 minutes; `-j` is half
   the core count, so ~32 jobs on the 7551P), then starts the model.
3. **Verify it** (section 4) before adding the second stack.
4. **Deploy `strata-coding`** the same way, with the shared values set
   identically in its own panel. It builds the same image (a layer-cache hit)
   and starts; stop it in Dockhand when you want the daily driver back.

Because the engine is baked into the image, container starts never compile. A
healthy start logs `engine already built for this PC` and goes straight to
loading. If you ever see it compiling at start, the image's `BUILD.json` did
not match — check that the build ran on this same host.

## 4. Verify

```sh
# Render GID still 107 before trusting group_add (compose asserts it exists)
stat -c '%n gid=%g' /dev/kfd /dev/dri/renderD*

# Host port 27182 is actually free of the TrueNAS UI
ss -lntp | grep -E ':(8080|27182)\b'

# GPU visible inside the container
docker exec strata-daily /opt/strata/engine/strata-device --list-devices

# Both mounts landed where intended
docker exec strata-daily sh -c 'ls /data; ls /data/models'

# The engine reused the baked build rather than recompiling
docker logs strata-daily 2>&1 | grep -i "already built\|compiled"

# Which hipBLASLt tuning table setup applied (want gfx1201 / 100202 on 7.2.4)
docker logs strata-daily 2>&1 | grep -i hipblaslt
```

The health check is the image's own (`/health`, 600 s start period), so
`docker ps` shows `healthy` once the model is loaded, and `GET /v1/status`
reports what is running.

## 5. Day-2

- **Switch profiles:** stop `strata-daily`, let it exit (its full 60 s
  `stop_grace_period`), then start `strata-coding`. Do not overlap them — a
  start while the previous engine still holds VRAM reads a stale free-VRAM
  figure and sizes the expert cache wrong.
- **Before starting ComfyUI:** stop whichever Strata profile is running.
- **Bump the engine:** change `STRATA_REF` in **both** stacks' env panels to the
  same newer tag, then Deploy each with "build images" enabled.
- **Swap the daily model:** set `STRATA_DAILY_FAMILY=qwen` for the base model at
  the same IQ2_XS size, or move to `IQ3_XXS`/`IQ3_S` for more quality at less
  speed (those need 43–50 GB of experts against a ~23.4 GiB card cache, so more
  work falls to the CPU). Changing settings for an already-configured model
  needs `REINSTALL=1`; switching to a model already on the volume does not.

## 6. Outstanding follow-ups

- **Benchmark your own numbers.** Every R9700 figure quoted for this build was
  measured on PCIe 4.0/5.0 platforms. The H11SSL-i is **PCIe 3.0**, so the GPU
  link is Gen3 x16 (~16 GB/s) and expert streaming is correspondingly slower.
  Run Strata's benchmark once the stack is up and replace the estimates.
- **Q8_0 vision encoder:** the docs recommend a Q8_0 mmproj for `--vision cpu`
  (590 MiB vs 865 MiB, less RAM, faster at the default 300 tokens, cosine
  0.997–0.999). It is **not hosted in either HF repo** and must be built with
  llama.cpp (`convert_hf_to_gguf.py --mmproj --outtype q8_0`, or
  `llama-quantize`). Deferred deliberately: BF16 works, and this is a
  size/RAM optimisation, not a correctness one.
- **UniFi integration:** for camera frames, raise the encoder's minimum image
  tokens (`"min_tokens": 1024` in the `"vision"` block of
  `strata-<tag>.json`) — llama.cpp's mtmd notes Qwen-VL wants at least 1024 for
  grounding tasks such as counting small items. The CPU default is 300 with no
  minimum.
