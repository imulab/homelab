# Strata deployment runbook

One Strata endpoint on the TrueNAS box, now on Strata 0.2.17 (engine 0.1.63) with the published
`ghcr.io/niko1221/strata:0.2.17-rocm-gfx120x` image. It serves Qwen3.8-Flash-Next Swift 1.5 IQ3_S at
262144 context on one Radeon AI PRO R9700 (RDNA4, gfx1201) in Docker. The engine ships in the image and the model shards are pre-seeded on the Optane, so the first
start downloads only the MTP drafter. All inference state lives in the 251 GiB of RAM.

| | |
|---|---|
| Model | Swift 1.5 IQ3_S |
| Context | 262144 |
| KV | int8 |
| Vision | on |
| RAM | 251 GiB, no container cap |
| Image | ghcr.io/niko1221/strata:0.2.17-rocm-gfx120x |
| Engine | 0.1.63 |
| API key | mandatory |
| Auto-start | yes (restart: unless-stopped) |

Swift 1.5 is kept over the base model because it thinks shorter, which suits the generalist plus
execution-only coding delegation role. IQ3_S is the quality tier. Upstream calls it "best; matches the
full model on the published tests". The 251 GiB of RAM removed the old memory ceiling. It did not remove
the decode bottleneck, which is the 32 GB VRAM expert cache and the CPU. The upstream
[DOCKER.md](https://github.com/Niko1221/Strata/blob/v0.2.17/docs/DOCKER.md) covers the settings and
[MODELS.md](https://github.com/Niko1221/Strata/blob/v0.2.17/docs/MODELS.md) covers the tiers.

## Upgrade from 0.1.41

This is a stack rewrite, not a ref bump. The 0.2.x image is prebuilt from the release engine package and
starts through `strata launch`. Nothing compiles at deploy. The old repo Dockerfile is deleted. The old
hand-built `strata-hip:local` image and the IQ3_XXS shards are still on the box for rollback.

The /data layout changed. `/data/Strata-data` holds the model files and the prepared pack. `/data/state`
holds presets, tuning and the key. `/data/install` holds run settings and logs. `/data/home` is new. On
first start the old `/data/models`, `/data/packs` and `/data/mtp` are moved into `/data/Strata-data`. The
old `/data/config` is not read.

The API key is now mandatory on a non-loopback bind. The old `STRATA_ALLOWED_HOSTS` variable is gone. The
old 64 GB container memory cap is removed on purpose. Upstream says the engine plans from host RAM and a
cap must exceed the plan or the start is OOM-killed. The stack sets `stop_grace_period: 2m`, unlimited
`memlock`, `shm_size: 2g`, host port 27182, container port 8080, and the external `proxy` network.

### Dockhand steps

1. Pull the stack source so the new `compose.yaml` is used.
2. Set the env panel values. `STRATA_API_KEY` is required (`openssl rand -hex 24`). `STRATA_MODEL=IQ3_S`.
   The rest are already present or defaulted. Keep the key-icon toggle OFF so the key is a plain env var.
3. Deploy.
4. Watch `docker logs -f strata`. Expect the MTP drafter download (5.2 GB, about 8 min at the measured
   11 MB/s) and then the model load from the Optane, about 10 to 15 min in total on the first start.
5. Verify with the commands in the Verify section.

### Rollback

The old image `strata-hip:local` and the old IQ3_XXS shards are still on the box. To go back, point the
stack at the old image and move the old packs and mtp back out of `/data/Strata-data`. The 0.1.x image
reads `/data/models`, `/data/packs` and `/data/mtp`, not `/data/Strata-data`.

### Measured on this box, 0.1.41, the comparison point

Measured here, 2026-11-21, on v0.1.41, Swift 1.5 IQ3_XXS, 262144 context, KV int8, vision on, warm cache,
one run per length, same repeated-text prompts. The 1K row includes first-request warm-up.

| Prompt | Read | Prefill | Generated | Decode |
| --- | ---: | ---: | ---: | ---: |
| 1086 tokens | 8642 ms | 125.7 tok/s | 234 | 39.6 tok/s |
| 8451 tokens | 9510 ms | 888.6 tok/s | 128 | 51.3 tok/s |
| 34062 tokens | 22282 ms | 1528.7 tok/s | 64 | 53.7 tok/s |

The expert cache held 15265 of 24576 slots, 99% hit (measured here).

Strata 0.2.17 on this box is not yet measured. These 0.1.41 numbers are the comparison point. Every
upstream number in this file is labelled upstream-measured.

### 0.2.17 engine defaults on this card

Upstream-measured or upstream-documented, from
[AMD_HIP.md](https://github.com/Niko1221/Strata/blob/v0.2.17/docs/AMD_HIP.md) and
[DETAILS.md](https://github.com/Niko1221/Strata/blob/v0.2.17/docs/DETAILS.md). On by default on gfx1201
Linux: matrix-core prompt attention (`STRATA_HIP_WMMA`), fused IQ3 prompt experts (`STRATA_PF_FUSED`), the
exact QSA scorer, 32K prefill chunks, pcie-frac 0.15, queue priming, and the n-gram table kept in RAM when
RAM allows. Upstream measured the n-gram table in RAM at +22.5% on an 8K prompt on an R9700.

Route tail skip is OFF by default because it changed answers. `STRATA_SET="STRATA_ROUTE_TAIL_SKIP=7"` opts
back in at a quality cost.

The published Linux rx9070 engine uses hipBLASLt 1.2.0 with no gfx1201 tuning table. Upstream measured 0 to
3% for that version, so the old hand-built 1.2.2 table is not being reproduced yet. That is the one open
performance question, settled by measurement.

## Model choice

Upstream-measured, from [MODELS.md](https://github.com/Niko1221/Strata/blob/v0.2.17/docs/MODELS.md).

### ISTA-DASLab tiers

| Size | RAM + VRAM | Quality |
| --- | ---: | --- |
| Q2_0 | 37.6 GB | fastest, good |
| IQ2_XS | 39.2 GB | fast, better |
| IQ3_XXS | 47.0 GB | slower, great |
| IQ3_S | 54.8 GB | slowest, best; matches the full model on the published tests |

IQ3_S is the pick. It holds about 50 GB of experts. It costs about 13% decode below IQ3_XXS, upstream-measured
on an RTX 5070 at 61.6 to 53.3 tok/s. The 251 GiB of RAM removed the old memory ceiling but not the decode
bottleneck. That bottleneck is the 32 GB VRAM expert cache and the CPU, and a bigger quant holds fewer
experts in VRAM.

Q8_0 now fits in RAM (~136 GB) but is slower and needs a 188 GB download, so it is not used.

### Unsloth 4 to 8 bit tiers

For the record. None of these is used here.

| Size | Download | Experts | RAM for all experts | Same top token as Q8_0 |
| --- | ---: | ---: | ---: | ---: |
| UD-IQ4_XS | 94 GB | 55 GiB | ~72 GB | 88.9% |
| UD-Q4_K_XL | 111 GB | 72 GiB | ~88 GB | 93.1% |
| UD-Q5_K_XL | 158 GB | 92 GiB | ~108 GB | 94.6% |
| UD-Q6_K_XL | 169 GB | 102 GiB | ~118 GB | 94.6% |
| Q8_0 | 188 GB | 120 GiB | ~136 GB | 100% (the reference) |

Upstream says IQ3_S matches the top token of Q8_0 87.7% of the time.

## Data layout

| Path | Contents |
| --- | --- |
| /data/Strata-data | model files, prepared pack |
| /data/state | presets, tuning, the key |
| /data/install | run settings, logs |
| /data/home | new in 0.2.x |

The shards are pre-seeded on the Optane at `/mnt/cistern/apps/strata/models/swift-IQ3_S`, mounted
read-only at `/models`, and `STRATA_MODEL_PATH=/models/swift-IQ3_S` names the folder that holds them
directly. The picture reader sits in the same folder, so `VISION=on` needs no download. A cold start reads
88.8 GB from the Optane in about a minute. Runtime speed does not depend on the mount, because the engine
copies the experts and the n-gram table into RAM at start.

The pre-seed is pinned to Hugging Face revision `99bb8f7f95c7aa7b24a36a7786a4f657b30f5d3d`:

```sh
MODELS=/mnt/cistern/apps/strata/models/swift-IQ3_S
REV=99bb8f7f95c7aa7b24a36a7786a4f657b30f5d3d
REPO=https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$REV
mkdir -p "$MODELS"
curl -fL -C - --retry 3 -o "$MODELS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf" \
  "$REPO/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf"
curl -fL -C - --retry 3 -o "$MODELS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf" \
  "$REPO/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf"
cp /mnt/cistern/apps/strata/models/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf "$MODELS/"
```

Expected sizes are 44,900,816,736 and 38,842,920,384 bytes. To download instead of pre-seeding, set
`STRATA_MODEL_PATH` empty before Deploy; the container then fetches about 89.7 GB into `/data` on first
start.

## Access

The API key is the whole access control. Every /v1 request needs `Authorization: Bearer <key>`. Only
GET /health is open without it. A key switches the DNS-rebinding host check off entirely, so
`STRATA_ALLOWED_HOSTS` is gone from this stack.

The Caddy label `caddy.reverse_proxy.header_up` injects the key for the hostname route, because the web UI
shares the API's origin and its own /v1 calls would 401 without it. Direct LAN clients on host port 27182
need the key too. If Hermes is re-enabled, its model config needs the same key in `api_key`, or every
request gets a 401.

Set the key as a plain env var in Dockhand, never the key-icon secret toggle. Secrets do not survive a
daemon-initiated restart, and this stack is `restart: unless-stopped`.

## Verify

```sh
docker ps
```

`healthy` means the model is loaded. `starting` means it is still downloading or loading.

```sh
docker logs strata 2>&1 | grep -iE "expert cache|slots|PLE|hipblaslt|pcie|WMMA|fused|matrix"
```

Read the expert cache fill line and its slot count. Read the PLE/table line. Read the hipBLASLt tuning
table line, which in 0.2.17 says no gfx1201 table ships. Read the pcie-frac default and the WMMA and fused
banners.

```sh
# Health is the one keyless endpoint
curl -s http://192.168.20.11:27182/health

# Models need the key
curl -s -H "Authorization: Bearer $STRATA_API_KEY" http://192.168.20.11:27182/v1/models

# One snapshot for scripts
docker exec strata strata monitor --once

# Clean stop, then clean start
docker stop strata
docker start strata
```

A clean start reuses the saved preset and is serving again in about a minute. `docker stop` runs
`strata launch --stop`, which drains and frees the graphics memory. The 2m grace period covers it.

## Benchmark

0.2.17 on this box is unmeasured. Before accepting any upgrade claim, repeat the 0.1.41 baseline with the
same three prompt lengths and the same repeated-text prompts.

Send one prompt at each of three lengths: 1086, 8451 and 34062 tokens. Read the split line from the engine
log.

```sh
docker logs strata 2>&1 | grep "strata serve: prompt"
```

The line reads `prompt P tokens = C reused + N read in R ms (X tok/s), G generated in M ms (Y tok/s)`.
X is prefill and Y is decode. Compare against the table in the Upgrade section. The 1K row there includes
first-request warm-up.

`TUNE=quick` is the sanctioned tuning path once a baseline exists. `TUNE=quick RETUNE=1` tunes again after
a model or engine change. The result is kept in the preset.

## Day-2 and outstanding

- **ComfyUI exclusivity.** This stack and comfyui both want the one GPU. Stop one in Dockhand before
  starting the other.
- **Vision toggle.** `VISION=on` uses the card's reader, about 1.4 GiB of VRAM and 0.1 to 0.5 s per picture.
  `VISION=off` removes image input and frees that VRAM.
- **Engine bump.** Change `STRATA_IMAGE_TAG` in the env panel, then Deploy. Pin the tag. Do not drift on
  `latest-rocm-gfx120x`. Check the [release page](https://github.com/Niko1221/Strata/releases/tag/v0.2.17)
  first.
- **Route tail skip.** Off by default because it changed answers. `STRATA_SET="STRATA_ROUTE_TAIL_SKIP=7"`
  turns it back on for faster decode on some cards, at the quality cost upstream now declines by default.
- **Open hipBLASLt question.** The published Linux rx9070 engine uses hipBLASLt 1.2.0 with no gfx1201
  tuning table. Upstream measured 0 to 3% for that version, so the old hand-built 1.2.2 table is not being
  reproduced yet. This is the one open performance question, settled by measurement.
- **Delete the old IQ3_XXS shards** from the Optane after the 0.2.17 stack is verified. Keep them until
  then for rollback.
- **Download instead of pre-seed.** Set `STRATA_MODEL_PATH` empty before Deploy and the container fetches
  about 89.7 GB into `/data`. The image's 2 hour health window covers it.
- **Hermes key.** If Hermes is re-enabled, set the same key in its `api_key` field.
