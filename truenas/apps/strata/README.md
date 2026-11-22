# Strata deployment runbook

One Strata endpoint on the TrueNAS box, now on Strata 0.2.17 (engine 0.1.63) with the published
`ghcr.io/niko1221/strata:0.2.17-rocm-gfx120x` image. It serves the abliterated Swift 1.5 IQ3_S (SC117) at
262144 context on one Radeon AI PRO R9700 (RDNA4, gfx1201) in Docker. The engine ships in the image and the model shards are pre-seeded on the Optane, so the first
start downloads only the MTP drafter. All inference state lives in the 251 GiB of RAM.

| | |
|---|---|
| Model | Swift 1.5 IQ3_S, abliterated (SC117) |
| Context | 262144 |
| KV | int8 |
| Vision | on |
| RAM | 251 GiB, no container cap |
| Image | ghcr.io/niko1221/strata:0.2.17-rocm-gfx120x |
| Engine | 0.1.63 |
| API key | mandatory |
| Auto-start | yes (restart: unless-stopped) |

The deployed build is the uncensored SC117 abliteration of the same Swift 1.5 GSQ-RCO IQ3_S, so the model
choice below is unchanged and the two builds are interchangeable through `STRATA_MODEL_PATH`.

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

The old image `strata-hip:local` is still on the box, and the IQ3_XXS shards are archived at
`/mnt/puddle/apps/strata/models-archive/swift-IQ3_XXS` because the Optane was needed for the abliterated
pre-seed. To go back to the old engine, copy them to `/mnt/cistern/apps/strata/models/swift-IQ3_XXS`, point
the stack at the old image, and move the old packs and mtp back out of `/data/Strata-data`. The 0.1.x image
reads `/data/models`, `/data/packs` and `/data/mtp`, not `/data/Strata-data`. To go back only to the censored
Swift build, set `STRATA_MODEL_PATH=/models/swift-IQ3_S`; those shards stay on the Optane until the
abliterated build is verified.

### Measured on this box, 0.1.41, the comparison point

Measured here, 2026-11-21, on v0.1.41, Swift 1.5 IQ3_XXS, 262144 context, KV int8, vision on, warm cache,
one run per length, same repeated-text prompts. The 1K row includes first-request warm-up.

| Prompt | Read | Prefill | Generated | Decode |
| --- | ---: | ---: | ---: | ---: |
| 1086 tokens | 8642 ms | 125.7 tok/s | 234 | 39.6 tok/s |
| 8451 tokens | 9510 ms | 888.6 tok/s | 128 | 51.3 tok/s |
| 34062 tokens | 22282 ms | 1528.7 tok/s | 64 | 53.7 tok/s |

The expert cache held 15265 of 24576 slots, 99% hit (measured here).

Strata 0.2.17 on this box was preflighted once on 2026-11-22, below. It is not benchmarked yet. Every
upstream number in this file is labelled upstream-measured.

### Preflight on this box, 2026-11-22

Verification data, not a benchmark. Measured here, one run unless stated.

- The stack came up: engine 0.1.63, model id `swift-1.5-iq3_s`, 262144 context, API key enforced, vision
  reader on the card, shards read from `/models/swift-IQ3_S`, `/data` migrated to `Strata-data`, serving
  about 2 minutes after start once the volume was prepared.
- Start-log defaults: PLE table locked in RAM in 27.5 s, PCIe probe 14.3 GB/s giving pcie_frac 0.11,
  expert cache 10178 slots in 19.31 GiB, `--batch 4` slot sessions taking 3.79 GiB of VRAM from that
  cache, vision reader 956 MiB, fused int8 prompt kernels on, WMMA on.
- The first two requests after load are settling artifacts (decode 8.8 and 5.5 tok/s). Do not benchmark
  the first minute.
- Settled, fresh prompts: 1086 tokens read at 296.1 tok/s with 29.4 tok/s decode; 8451 at 1517.1 with
  28.9; 34062 at 1752.3 with 26.1.
- Cache-warm repeats decoded at 84.3, 75.6 and 97.3 tok/s. Two fresh 512-token generations decoded at
  51.8 and 25.0 tok/s. Decode is not stable in these short runs, so no decode claim is made here.

Against the 0.1.41 table above, prefill is ahead at 8K (+71%) and 32K (+15%). Decode needs a sustained
run after deploy. The first knob to test is `PARALLEL=1`, which gives the 3.79 GiB of batch-session VRAM
back to the expert cache; `TUNE=quick` is the sanctioned tuning path.

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

### Uncensored variant

The deployed shards are the abliterated counterpart of the same model:
[SC117/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF](https://huggingface.co/SC117/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF).
It is the same Swift 1.5 GSQ-RCO IQ3_S with the refusal direction removed from 144 residual-writing tensors
across all 48 layers, and the other 1,080 tensors are byte-identical to upstream. The repo's own measurements
say Swift's reasoning efficiency survives and the ablation costs about 2% decode on IQ2_XS. Strata sees an
ordinary GSQ-RCO pack, so no patched engine is needed.

Two Swift-specific traps from that repo apply. `--ple-gguf` must point at shard 1 because Swift keeps the PLE
table there, and the run config already does. The native pack must be rebuilt for the abliterated tensor
offsets, so delete `/data/Strata-data/packs/swift-iq3_s` before the first start with the new shards and let
setup rebuild it.

The start log reports the shard size and SHA-256 difference from the official Swift 1.5 files and uses them
anyway, because the own-file path accepts a complete GGUF as another version or fine-tune. That warning is
expected. Set `STRATA_MODEL_PATH=/models/swift-IQ3_S` and delete the pack again to switch back to the
censored build.

License: the Swift Open License v1.0 and the Qwen Community License carry over. This is an independent
community build, not affiliated with UkisAI or Strata.

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

The shards are pre-seeded on the Optane at `/mnt/cistern/apps/strata/models/sc117-swift-IQ3_S`, mounted
read-only at `/models`, and `STRATA_MODEL_PATH=/models/sc117-swift-IQ3_S` names the folder that holds them
directly. The picture reader sits in the same folder, so `VISION=on` needs no download. A cold start reads
83.5 GB from the Optane in about a minute. Runtime speed does not depend on the mount, because the engine
copies the experts and the n-gram table into RAM at start.

The pre-seed is pinned to Hugging Face revision `3db919a964b1f0ae0d1000ba7be90b3e38b1f7a3`:

```sh
MODELS=/mnt/cistern/apps/strata/models/sc117-swift-IQ3_S
REV=3db919a964b1f0ae0d1000ba7be90b3e38b1f7a3
REPO=https://huggingface.co/SC117/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-GGUF/resolve/$REV
mkdir -p "$MODELS"
curl -fL -C - --retry 3 -o "$MODELS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ3_S-00001-of-00002.gguf" \
  "$REPO/Swift-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ3_S-00001-of-00002.gguf"
curl -fL -C - --retry 3 -o "$MODELS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ3_S-00002-of-00002.gguf" \
  "$REPO/Swift-Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ3_S-00002-of-00002.gguf"
cp /mnt/cistern/apps/strata/models/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf "$MODELS/"
```

Expected sizes are 44,849,330,016 and 38,699,929,024 bytes. Both SHA-256 values were verified against the
Hugging Face LFS pointers on 2026-11-22. To download instead of pre-seeding, set `STRATA_MODEL_PATH` empty
before Deploy; the container then fetches about 89.7 GB into `/data` on first start.

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
