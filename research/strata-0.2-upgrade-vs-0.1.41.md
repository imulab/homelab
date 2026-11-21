# Strata 0.2.x vs the deployed v0.1.41 stack

> Compiled 2026-10-20 via `/research`, against the Strata repository (tags, release notes, `docs/DOCKER.md`, `docs/AMD_HIP.md`, `setup.py`, `serve/server.py`, `docker/entrypoint.sh`), the GitHub Container Registry, and this repo's own runbook and measurements.
> Target: **one** AMD Radeon AI PRO R9700 (RDNA4, gfx1201, 32 GB) in Docker on TrueNAS SCALE 26, EPYC 7551P, **PCIe 3.0 x16**, 128 GB RAM, Optane 900P for the model shards. Incumbent: `truenas/apps/strata`, `STRATA_REF=v0.1.41`, Swift 1.5 **IQ3_XXS** at 262144 context, `KV=int8`, `VISION=cpu`, measured **28.6 tok/s** sustained decode.
> Companion to `truenas/apps/strata/README.md` (the runbook, whose §9 already names v0.1.43 as the next step) and `research/radiance-vs-strata-rdna4.md` (the "stay on Strata" engine survey).
> **Superseded 2026-11-21.** The hold decision below was overturned and the stack was upgraded. The 128 GB RAM figure in the target line above is also superseded; the box now has 251 GiB. The current decision, the 0.2.17 evidence and the new baseline follow.

## Decision update (2026-11-21): upgraded to v0.2.17, engine 0.1.63

**Taken: Strata v0.2.17 with the published `ghcr.io/niko1221/strata:0.2.17-rocm-gfx120x` image, Swift 1.5 IQ3_S, 262144 context, KV int8, vision on, mandatory API key.** The hold conditions did not literally fire; the environment changed instead.

What changed since the 2026-10-20 hold:

- **The release line settled.** 0.2.x went from three tags to a full line; 0.2.17 shipped 2026-11-20 with engine 0.1.63. The published `rocm-gfx120x` image exists for 0.2.17 (registry manifest 200), so the migration is now a pull plus an env panel, not a source build.
- **The gfx12 wins that 0.1.41 lacked are now defaults in the shipped image.** Matrix-core prompt attention, fused IQ3 prompt experts, the exact QSA scorer, 32K prefill chunks, `--pcie-frac` 0.15 and queue priming are all on, with no `STRATA_REF` rebuild to reach them. Sources: `docs/AMD_HIP.md` and `docs/DETAILS.md` at v0.2.17.
- **The n-gram table now goes to RAM when the RAM allows (0.2.7 and later), and the box has 251 GiB.** On an R9700 that measured +22.5% on an 8K prompt (2,045 to 2,506 tok/s) and +9.2% at 32K. This removes the old reason the second shard had to be read from the Optane during inference. The shards are pre-seeded on the Optane anyway, so a cold start reads them in about a minute; runtime speed does not depend on it, because the engine copies the experts and the n-gram table into RAM at start.
- **The 256 GB upgrade settles the model question in the opposite direction from the assumption that started it.** RAM is no longer the constraint, so the quant is a pure quality against decode trade, and the 32 GB VRAM expert cache plus the CPU bound decode. IQ3_S is the tier whose quality upstream calls "matches the full model on the published tests"; it costs about 13% decode against IQ3_XXS. Q8_0 now fits in RAM (~136 GB) but is slower again and needs a 188 GB download, so it is not used.
- **Prefill at 32K was measured on this box before the switch**, which the hold decision named as the prerequisite. See the baseline below. Prefill was the one axis the upgrade moves.

Baseline, measured here on 2026-11-21 on v0.1.41 (Swift 1.5 IQ3_XXS, 262144 context, KV int8, vision on, warm cache, same repeated-text prompts, one run per length):

| Prompt tokens | Read ms | Prefill tok/s | Generated | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: |
| 1,086 | 8,642 | 125.7 | 234 | 39.6 |
| 8,451 | 9,510 | 888.6 | 128 | 51.3 |
| 34,062 | 22,282 | 1,528.7 | 64 | 53.7 |

The 1K row carries first-request warm-up. Expert cache was 15,265 of 24,576 slots at 99% hit. These are the comparison point; **v0.2.17 on this box is not measured yet**, and no 0.2.17 speed claim is made here until it is.

Open after the switch:

- **The Linux `rx9070` package uses hipBLASLt 1.2.0 with no gfx1201 tuning table.** Upstream measured tuning at 1.2.0 at 0 to 3%, within noise, so this may cost nothing. The hand-built 0.1.41 image used ROCm 7.2.4 with the 1.2.2 table, which was worth 638 to 1,177 tok/s at 32K prompt on an R9700. The 32K prefill comparison above settles whether a source build is needed. It was rejected for the first deploy as the larger change.
- **Route tail skip is off by default in 0.2.17 because it changed answers.** `STRATA_SET="STRATA_ROUTE_TAIL_SKIP=7"` opts back in, at a quality cost. Leave it off until there is a reason.
- **Tuning is per model and per box.** `TUNE=quick` measures once and saves the preset; `RETUNE=1` tunes again after a model or engine change. Do it after the baseline A/B, not during the first deploy.
- **Rollback is intact.** The old `strata-hip:local` image and the IQ3_XXS shards remain on the box. Rolling back also needs the old `/data/packs` and `/data/mtp` moved back out of `/data/Strata-data`, because 0.1.x does not read the new layout.

## Superseded decision (2026-10-20): hold — no engine change, 0.1.x or 0.2.x

**Deployed stays `STRATA_REF=v0.1.41`. No interim bump to 0.1.45, no migration to 0.2.x yet.** Two reasons, and the first is the one that actually settles it:

- **0.1.45 is a dead end; not taking it is correct.** It buys only interim speed, and every one of its gains is *contained in* whatever 0.2.x release we eventually land — so doing it means paying for a full HIP rebuild and a re-verification pass on an engine that gets replaced, not upgraded, by the next step. Skipping it costs nothing at migration time.
- **No rush, by precedent.** 0.1.x ran ~45 tags from `v0.1.0` to `v0.1.45` before this stack was installed at `v0.1.41`; 0.2.x is at `0.2.1.1` — three tags in. 0.2.1 already exists mainly to fix a pile of 0.2.0-beta defects (crashes, stuck downloads, card-selection, re-download on unload), which is the same pattern that made the wait worthwhile on 0.1.x. The alignment motive stands — 0.2.x *is* the current model, and the switch is coming — but it is worth paying once, against a settled release, and not under time pressure.

**The switch is a rewrite of the stack, not a bump** (§3, §5): new packaging model, `/data/Strata-data` layout, mandatory API key, and `STRATA_ALLOWED_HOSTS` made inert. That is the work being deferred, and it is deferred as one unit on purpose — half-migrating is strictly worse than either end state.

### What re-opens this

Not a date. Any one of these:

- **A 0.2.x release ships a gfx1201 hipBLASLt table for its *packaged* engine.** This is the single change that would remove the main cost of the easy path (§3, Step 2): today the only `ghcr` image for this card runs plain hipBLAS, so we would still have to build from source to keep the `100202` table's 638 → 1,177 tok/s at 32K. Fix that and the migration becomes a pull, a data-layout move and a key.
- **0.2.x fixes something we actually hit**, rather than something we might: the R9700 queue-prime fix is already in 0.2.1 and is the nearest candidate, since our stack restarts itself on every reboot by design.
- **We need the AMD GPU vision reader** (57 ms vs 3.1 s per 260-token image) for OCR or vault indexing at a volume where CPU encoding is the bottleneck.
- **`docker stop`/start behaviour, or the `REINSTALL=1` config-regeneration hazard, bites us in production** — i.e. we lose a hand-set option to a redeploy and have to care.

Two things to do *before* any of these fire, because they are cheap now and load-bearing later: **measure prefill at 32K** (still this runbook's largest unknown and the only axis any upgrade claim moves), and **record the current args array** from `strata-swift-iq3_xxs.json` so the hand-set `--pcie-frac`/`--vram-reserve-mib` values survive the layout move into `user_args`.

## TL;DR

- **There is no released "0.2.0".** The 0.2 line opened as **`v0.2.0-beta`** on 2026-10-19 and was replaced **the next day** by **`v0.2.1`** (2026-10-20), then **`v0.2.1.1`** hours later. The only tag matching "0.2.0" is that beta. `0.2.1`'s own notes say it "fixes the crashes, stuck downloads and GPU-choice bugs reported on 0.2.0-beta", and that it *replaces* several of the beta's defaults. **Any plan built on "0.2.0" is a plan built on a superseded prerelease** — the target, if this is done at all, is **0.2.1**, whose engines are also unchanged by 0.2.1.1 (Terminal-only fix).
- **We are nine refs behind, not one release behind.** Deployed: `v0.1.41`. Since then: 0.1.42, 0.1.42.1, 0.1.43, 0.1.43.1, 0.1.44, 0.1.44.1, 0.1.45, then 0.2.0-beta, 0.2.1. Crucially the **RDNA4 (gfx1200/gfx1201) prompt + decode defaults landed at 0.1.43, not 0.2.0** — route tail skip, the measured PCIe share, 16K prompt chunks and fused IQ3 prompt experts are all `v0.1.43` defaults. So most of what "upgrading to 0.2.0" sounds like is actually capturable by a `STRATA_REF` bump.
- **Expected decode (tok/s) improvement is small — single-digit to low-double-digit percent, not a multiplier.** The documented R9700/IQ3_XXS comparison for 0.1.43 is **decode +2% to +10%, prompt +12% to +14%**. 0.1.45's large decode win (**83.2 → 117.8 tok/s, 1.42x**) is measured on **UD-Q4_K_XL**, a different quant, and is *not* claimed for IQ3_XXS. Every 0.2.x gfx12 measurement states decode is **untouched**: `0.999`/`1.000`/`1.001`. There is no documented path from 28.6 tok/s to 50–60 tok/s.
- **The 50–60 tok/s figure is not a decode measurement on this box.** The running config's only sustained number is **28.6 tok/s** (3,280 tokens in 119 s). Strata's own docs call the web UI rate "a rate over a window, not a mean since the first token", and the runbook records UI bursts near 80 tok/s. Separately, **`reasoning_effort` is not a throughput control at all** — it is per-request prompt steering (`none | low | medium | high | xhigh`, default `high`) that changes *how many thinking tokens the model emits*, i.e. time-to-answer, not tokens/second. A thinking level cannot appear in a tok/s delta.
- **The upgrade is not "just a docker compilation".** 0.2.x **replaced the whole Docker packaging model**: the image is no longer compiled from a pinned source checkout at `docker build`; it is assembled from a signed, prebuilt **engine package**, and the container now starts through **`strata launch`** instead of `setup.py` + `docker-entrypoint.sh`. `docker-entrypoint.sh` does not exist at v0.2.1. The `/data` layout changed too (`/data/Strata-data`, `--data-dir`), the old `/data/models` + `/data/config` layout **is explicitly not read**, and an **API key becomes mandatory** because the server hard-refuses a non-loopback bind without one.
- **Recommended path: hold, then go straight to a settled 0.2.x — skip 0.1.45.** `STRATA_REF=v0.1.45` *would* be a drop-in rebuild for the Dockerfile we already have (`docker-entrypoint.sh` and the `/data` layout are unchanged; `requirements.txt`/`setup.py` contract identical), with no re-seed, no re-download, the keyless topology intact and the gfx1201 hipBLASLt table preserved — **but it is rejected on purpose**: it buys only interim speed, and its gains are contained in any later 0.2.x anyway, so taking it means rebuilding and re-verifying an engine that gets replaced rather than upgraded. See the Decision section. The 0.2.x migration itself is §3 Step 2 against the §5 checklist, and its blockers are the mandatory API key and the `/data` layout move.

## 1. What actually shipped, in order

| Tag | Date | Engine package | What it is |
|---|---|---|---|
| `v0.1.41` | 2026-10-08 | 0.1.41 | **deployed** |
| `v0.1.43` | 2026-10-11 | 0.1.43 | **RDNA4 defaults**: route tail skip, measured PCIe share, 16K prompt chunks, fused IQ3 prompt experts |
| `v0.1.45` | 2026-10-17 | 0.1.45 | new decode kernels, bigger default prompt chunk, multi-request serving |
| `v0.2.0-beta` | 2026-10-19 | 0.1.45.3 | app + Terminal + engine split; gfx12 WMMA/fused defaults; GPU vision reader; **crash/stuck-download bugs** |
| `v0.2.1` | 2026-10-20 | 0.1.46 | the fixes for those bugs, task board, `strata bench run`, **published Docker images**, `user_args` |
| `v0.2.1.1` | 2026-10-20 | unchanged | Windows Terminal long-path fix only; **no image published** (`0.2.1.1-*` is 404 on ghcr) |

`setup.py` in both 0.1.45 and 0.2.1 sets `MIN_ENGINE = (0, 1, 45)`, so a compiled 0.1.41 engine can never be reused by newer source — **every one of these steps is a rebuild**, which is why the size of the rebuild matters more than the size of the version jump.

## 2. Where the gains actually are, and on which tier

All figures below are the project's own, on an R9700 unless marked otherwise. They are *not* transferable to this box's Zen 1 CPU without measuring, which is exactly the correction the runbook already had to make once (60–90 tok/s projected, 28.6 measured).

| Change | Tier measured | Decode | Prompt |
|---|---|---|---|
| 0.1.43 RDNA4 defaults | **IQ3_XXS**, R9700 32 GB | **+2% … +10%** | **+12% … +14%** |
| 0.1.43 RDNA4 defaults | Q2_0, R9700 32 GB | +2% … +10% | +12% … +14% |
| 0.1.45 decode kernels | **UD-Q4_K_XL** only, R9700 | **1.42x** (83.2 → 117.8) | 4K **1.80x** |
| 0.1.45 long-prompt AMD kernels | R9700 | — | 131K **+8.7%**, 230K **+16%**, 500K **+79%** |
| 0.2.0-beta gfx12 prompt defaults (WMMA + fused) | **IQ3_XXS**, R9700 | **unchanged (0.999)** | 4K **1.05x**, 20K **1.27x** |
| 0.2.0-beta gfx12 prompt defaults | Q2_0, R9700 | unchanged | 4K **1.99x**, 20K **1.99x** |
| 0.2.1 GPU-queue prime | R9700, Linux | **1.000 / 1.001 / 1.000** | removes a **8–23%** start-up lottery |
| 0.2.1 IO prefetch + decode gate | **RTX 3060, `experts.bin` pack** | **1.21x … 1.38x** | 1.04x–1.06x |

Three things this table is telling us:

1. **The big prompt multipliers on 0.2.x are Q2_0, not our tier.** At IQ3_XXS the 0.2.0-beta prompt gain is **1.05x at 4K and 1.27x at 20K** — respectable at depth, modest at short context. The eye-catching 1.99x belongs to Q2_0.
2. **0.2.1's only genuine decode multiplier does not apply to our pack.** It is gated on `experts.bin` packs read from page cache with `--mmap-experts`; our pack is a *native IQ pack* that indexes tensors and leaves the experts in the shard (`tools/iq_pack.py`). It was also measured on a 12 GB CUDA card, not on gfx1201 + Optane.
3. **0.2.1's R9700 change is reliability, not throughput.** Its own measurement is decode `1.000`. What it removes is a known R9700 defect — *"about half of the fresh engines with MTP read prompts 10–23% slower than the others"* — which on a `restart: unless-stopped` stack is a lottery paid on every reboot.

## 3. The three upgrade paths, costed

### Step 1 — `STRATA_REF=v0.1.45` (available, and rejected — see the Decision section)

- **Build:** unchanged Dockerfile. `docker-entrypoint.sh` still exists at v0.1.45 and diffs against v0.1.41 by one added AVX2 warning block; `/data/models`, `/data/config`, `REINSTALL`, `LOW_RAM`, `VISION`, `KV`, `CONTEXT` all keep their meaning. The 10–20 minute HIP compile as usual.
- **Re-seed:** **no.** Same directory tags (`models/swift-IQ3_XXS`, `config/strata-swift-iq3_xxs.json`, `packs/swift-iq3_xxs`), same validation.
- **Downloads:** **none.** Pack rebuild is seconds and local; the MTP layer is already on the volume.
- **hipBLASLt table:** **kept, automatically.** `setup.py` at 0.1.45 still calls `hipblaslt_table(arch, lib_dirs, ver)` and auto-selects `tools/hip/gfx1201-hipblaslt-100202.txt` because the base image is ROCm 7.2.4 / hipBLASLt 1.2.2. That is the pairing the Strata docs credit with **638 → 1,177 tok/s** on a 32K prompt.
- **Config caveat:** the new 0.1.43+ prompt defaults are *presets*, and `setup.py` regenerates the config when a preset it owns changes — so a hand-added `--pcie-frac` may be dropped. Re-read the args array after first start rather than assuming.
- **Does not give:** `user_args` protection, keyless-with-`STRATA_ALLOWED_HOSTS` surviving, the benchmark suite, the GPU vision reader, or a compile-free start.

### Step 2 — 0.2.1 built from source on the same base (the only 0.2.x variant worth taking)

- **Why not the ready-made image first.** `ghcr.io/niko1221/strata:0.2.1-rocm-gfx120x` exists (verified: manifest 200; `latest-rocm-gfx120x` too) and would remove the compile entirely — 2.8 GB on disk, ~505 MB pull, start-up tested on an actual R9700. **But the packaged Linux `rx9070` engine ships no gfx1201 hipBLASLt tuning table**: its bundled hipBLASLt is 1.2.0, and the only gfx1201 tables in the tree are for 1.2.2 (`100202`) and 1.5.0 (`100500`). On gfx1201 the project's own note is that the *plain* path is "far off" — the table is what recovers ~1.8x at 32K. Pulling the image therefore trades the runbook's single largest measured prompt win for the 0.2.x prompt defaults, which are worth far less at our tier. **Build from source on `rocm/dev-ubuntu-24.04:7.2.4-complete` and keep the table.**
- **What the rebuild costs.** The Dockerfile must be rewritten: no `docker-entrypoint.sh`, and the container starts `strata launch --no-tui --access lan --port $PORT --data-dir /data/Strata-data`. The image must also carry the **Terminal** bundle (`/opt/strata/terminal/bin/strata`) that `strata launch` needs, plus llama.cpp at the pinned commit. The documented from-source route for AMD is now three steps: build the engine (`docs/AMD_HIP.md`) → `packaging/package_engine.py` → `docker/build.sh --flavor hip --engine <pkg>`. A hand-rolled Dockerfile that still calls `setup.build_engine_hip()` will not produce the image the new entrypoint expects.
- **Re-seed:** **no re-download of the 76 GB**, but the layout is new. `MODEL_PATH` pointing at the existing `/data/models/swift-IQ3_XXS` is the documented escape hatch ("nothing is downloaded for the model itself"), and the mount stays the Optane, so shard 2 is still flash-resident. The **vision encoder is a separate file one directory up** (`/data/models/mmproj-*.gguf`) — check whether it must move into the model folder for `VISION`.
- **Downloads:** the **MTP draft layer is re-fetched** (~5 GB, into `/data/Strata-data/mtp/rt`) unless the old `/data/mtp` artefacts are moved there by hand. Downloads are ~1.9x faster in 0.2.1 for the model files (16 connections vs 8), so this is cheaper than it was. The pack is **remade in the volume** at `/data/Strata-data/packs/`.
- **Breaking for our topology:** see §5 — the mandatory API key is the big one.

### Step 3 — do nothing

Defensible. v0.1.41 works, is measured, and the engine is baked into the image so starts never compile. The cost of waiting is bounded to prompt throughput at depth.

## 4. So what is the expected improvement?

Stated honestly, per axis, for **our** box and **our** tier:

- **Decode (the number the user is quoting): ~0–15%.** Composed of 0.1.43's `+2…+10%` on IQ3_XXS, plus an unquantified share of 0.1.45's new decode kernels (measured on a different quant, so treat as unknown), plus **0%** from 0.2.x, which states decode unchanged on gfx12. **28.6 tok/s → roughly 29–33 tok/s.** It does not become 50–60.
- **Prefill/prompt: the real win, and it is larger than one release suggests.** 0.1.43's `+12…+14%`, 0.1.45's long-prompt kernels (`+8.7%` at 131K, `+16%` at 230K), and 0.2.x's `1.05x`/`1.27x` at 4K/20K compound toward something in the **+25% to +45%** band at depth — but this is the one dimension the runbook flags as **never measured on this box** (every recorded request was a short prompt with a long answer). It is the number to establish *before* accepting any upgrade claim.
- **Ops/UX wins, if 0.2.1 is taken:** no 10–20 minute HIP compile per deploy (or a 2.8 GB image pull); `user_args` so the engine no longer overwrites hand-set options like `--vram-reserve-mib` and `--pcie-frac`; a fixed `standard-v1` benchmark suite (34 request files, scorer, validator) which is precisely the instrument this runbook has been missing; AMD GPU vision at **57 ms vs 3.1 s** per 260-token image; model downloads ~1.9x faster.
- **Regression risk:** the mandatory API key (§5), the data-layout migration, and — if the ready-made image is taken — losing the hipBLASLt table.

## 5. Breaking changes to schedule before any 0.2.x work

1. **The API key is now mandatory.** `serve/server.py` refuses to start a non-loopback bind without one — *"a non-loopback bind requires an inference API key"*, exit 2 — and the new entrypoint always launches with `--access lan`. A key is either supplied (`API_KEY` / `API_KEY_FILE`) or generated on first start into `/data/state/api-key`, and **every `/v1` request must carry `Authorization: Bearer <key>`** (`/health` excepted). Our stack is keyless *by design*; Hermes, Caddy, and any client would each need the key, and the Caddy route needs `header_up` because the web UI shares the API's origin.
2. **`STRATA_ALLOWED_HOSTS` becomes inert.** The Host check still exists, but it only runs when no key is set (`if svc.api_key or host_allowed(...)`). Setting the now-mandatory key switches the whole DNS-rebinding path off, so the carefully-documented `strata.absurdlab.dev,strata` fix — and the runbook section defending it — becomes historical.
3. **`/data` layout.** `--data-dir /data/Strata-data`; the old `/data/models` and `/data/config` are **not read**. `/data/state`, `/data/install`, `/data/home` are new.
4. **Ignored env vars.** `LOW_RAM`, `RESIDENT_BUDGET_GIB`, `KV_STREAMING`, `LAYER_SPLIT`, `CONFIG`, `REINSTALL` are accepted and **ignored with a warning**. `GGUF_DIR` maps to `MODEL_PATH`. Engine settings move to `STRATA_SET` (`"prefill=65536 STRATA_PIN_BY_NEED=1"`). `VISION` gains `off | on | gpu | cpu` (`no`/`yes` still accepted).
5. **Healthcheck and stop.** `/v1/models`-with-key is the new documented health check (`/health` still answers keyless, so our `curl /health` healthcheck survives). `docker stop` now runs `strata launch --stop`; the documented grace period is **120 s**, double our current 60 s.
6. **`MIN_ENGINE = (0,1,45)`** — a partially-upgraded tree cannot reuse the old compiled engine, and a source build at 0.2.1 reports engine 0.1.46.

## 6. Confidence notes

- **Primary-source (fetched 2026-10-20):** the full tag list (54 tags; no `v0.2.0`); published-at dates for `v0.2.0-beta`, `v0.2.1`; every quoted measurement from the 0.1.43 / 0.1.45 / 0.2.0-beta / 0.2.1 release bodies; `v0.2.1`'s `docs/DOCKER.md` (image tags, settings table, "from the older image" mapping, "the old volume layout is not read"); `docs/AMD_HIP.md` at v0.2.1 (§hipBLASLt: the 1.2.2 table and its 638 → 1,177 tok/s figure; "Linux `rx9070` has none"; `STRATA_HIPBLASLT_TABLE`); `setup.py` at v0.1.45 and v0.2.1 (`MIN_ENGINE`, `hipblaslt_table()`, the `tools/hip` table inventory); `serve/server.py` at v0.2.1 (the non-loopback key refusal at the `preflight_api_key` check; the `api_key or host_allowed` branch; `reasoning_effort` handling); `docker/entrypoint.sh` at v0.2.1 (`--access lan`, `--data-dir /data/Strata-data`, `MODEL_PATH`); the tree diff showing `docker-entrypoint.sh` present at v0.1.41/v0.1.45 and absent at v0.2.1; the entrypoint diff v0.1.41 → v0.1.45 (one AVX2 warning block); `docs/DETAILS.md` for the `reasoning_effort` levels.
- **Registry-verified:** `ghcr.io/niko1221/strata:0.2.1-rocm-gfx120x` and `:latest-rocm-gfx120x` resolve (HTTP 200); `0.2.1.1-rocm-gfx120x` does not (404). Tag list is `0.2.1-{cuda12,cuda13,rocm-gfx110x,rocm-gfx1151,rocm-gfx120x}` plus the `latest-*` equivalents.
- **Local (repo) sources:** `truenas/apps/strata/{README.md,Dockerfile,compose.yaml,.env.example}`, `truenas/apps/hermes/*`, `truenas/apps/README.md`, `research/strata-endpoint-api-surface.md`.
- **Undetermined — measure, do not assume:**
  - **Prefill on this box.** Still unmeasured at 32K+, which is the one phase every upgrade claim here moves. Any decision made without it is made on someone else's hardware.
  - **Whether the 0.1.45 decode kernels help IQ3_XXS at all.** Every published figure for them is UD-Q4_K_XL (or UD-Q6_K_XL); the IQ3_XXS row is absent.
  - **Whether a native IQ pack benefits from 0.2.1's IO-prefetch/decode-gate default.** The change is documented for `experts.bin` packs with `--mmap-experts` on CUDA; our pack leaves experts in the shard.
  - **Whether the old `/data/mtp` runtime files are format-compatible with 0.2.1's `/data/Strata-data/mtp/rt`.** Moving them would save the ~5 GB re-fetch; the tools (`mtp_fetch`/`mtp_pack`/`mtp_rt`) still ship, but compatibility is not stated.
  - **Whether a hand-rolled 0.2.1 image on ROCm 7.2.4 reproduces the table.** The logic is version-matched (`tools/hip/gfx1201-hipblaslt-100202.txt`), so it should, but the 0.2.1 release notes only measured the packaged engine's tables.
  - **Whether the 0.2.1 prompt defaults are additive with the 100202 table or overlapping with it.** Both act on prefill, by different mechanisms (attention vs dense GEMM); the combined figure on our box is unmeasured.

## Sources

[Strata releases](https://github.com/Niko1221/Strata/releases) · [v0.2.1](https://github.com/Niko1221/Strata/releases/tag/v0.2.1) · [v0.2.0-beta](https://github.com/Niko1221/Strata/releases/tag/v0.2.0-beta) · [v0.1.45](https://github.com/Niko1221/Strata/releases/tag/v0.1.45) · [v0.1.43](https://github.com/Niko1221/Strata/releases/tag/v0.1.43) · [tags API](https://api.github.com/repos/Niko1221/Strata/tags) · [DOCKER.md at v0.2.1](https://github.com/Niko1221/Strata/blob/v0.2.1/docs/DOCKER.md) · [AMD_HIP.md at v0.2.1](https://github.com/Niko1221/Strata/blob/v0.2.1/docs/AMD_HIP.md) · [DETAILS.md](https://github.com/Niko1221/Strata/blob/v0.2.1/docs/DETAILS.md) · [setup.py at v0.1.45](https://github.com/Niko1221/Strata/blob/v0.1.45/setup.py) · [docker/entrypoint.sh at v0.2.1](https://github.com/Niko1221/Strata/blob/v0.2.1/docker/entrypoint.sh) · [serve/server.py at v0.2.1](https://github.com/Niko1221/Strata/blob/v0.2.1/serve/server.py) · [ghcr.io/niko1221/strata](https://github.com/Niko1221/Strata/pkgs/container/strata) · local: `truenas/apps/strata/README.md` (§5–§9), `truenas/apps/strata/.env.example`, `research/strata-endpoint-api-surface.md`
