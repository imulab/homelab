# Strata — deployment runbook

One inference profile of **Qwen3.8-Flash-Next (Swift 1.5, IQ3_XXS)** at **256K
context** on the AMD Radeon AI PRO R9700 (RDNA4, gfx1201), served by
[Strata](https://github.com/Niko1221/Strata) over its HIP backend.

| | |
|---|---|
| Model | Swift 1.5 **IQ3_XXS** (full 512 experts, 3.0 bpw) |
| Context | **262144** (native, no rope scaling) |
| KV | `int8`, streamed (`--kv-resident 32768`) |
| Vision | `cpu` (OCR, NAS video-vault indexing) |
| RAM cap | 64 GB |
| Auto-start | **yes** (`restart: unless-stopped`) |

This replaced the earlier two-stack daily/coding split. That split existed
because Compose cannot express "defined but not started" for a single service
(verified against Docker: `restart: "no"` still starts on `up`, while
`deploy.replicas: 0` and `profiles:` leave no container at all, so Dockhand
could not start it later). With one model there is nothing to switch between, so
the second stack, its duplicate Dockerfile and the two hostnames all go away.
**`strata` and `comfyui` remain mutually exclusive** — one GPU, no VRAM
isolation between ROCm processes. Stop one in Dockhand before starting the
other; both publish a host port, so a mistaken second start also fails fast on
the bind.

## Model choice

### The tier

Resident expert slots are the main throughput lever on this card, and they
shrink as the quant grows. The slots below are **projections** from the measured
R9700 configuration in `docs/AMD_HIP.md` (12,288 slots in 23.4 GiB — ~1.90
MiB/slot for the Coder's IQ1_M) plus the larger per-slot cost of the bigger IQ
formats. Only the IQ3_XXS row has since been measured on this box, and it was
**wrong by a wide margin in the pessimistic direction** — see
[Measured on this box](#measured-on-this-box):

| Tier | expert arena | fits 900P | ≈ slots resident | share |
|---|---:|---|---:|---:|
| Q2_0 | 34.0 GB | yes | most | — |
| IQ2_XS | 35.5 GB | yes | ~14,000 / 24,576 | ~57% (projected) |
| **IQ3_XXS** | **42.9 GB** | yes | **15,271 / 24,576** | **62% (measured)** |
| IQ3_S | 50.3 GB | yes | ~8,600 / 24,576 | ~35% (projected) |
| Coder IQ1_M | 23.4 GB | yes | 12,288 / 12,288 | 100% |

The projection for IQ3_XXS said ~10,000 slots (41%). The card actually holds
15,271. Two things drove the underestimate, and only the first is certain from
the measurement: the per-slot cost was derived from Strata's Coder reference
(`IQ1_M`, 12,288 slots in 23.4 GiB ≈ 1.90 MiB/slot) and applied unchanged to
IQ3_XXS, whereas the measured IQ3_XXS figure is ~1.70 MiB/slot — the slot size
does not transfer between tiers; and the VRAM left for the cache was taken to be
smaller than it evidently is. Treat the remaining rows as unmeasured; only
IQ3_XXS is used here, and the other tiers would need their own measurement before
any of them is trusted.

IQ2_XS is the faster tier and IQ3_XXS the better one. The quality case for IQ3
here is not the benchmark average — it is *where* the 2-bit tier degrades first.
Independent reports on this model family agree that quantization erodes
instruction-following, output-format stability and stopping behaviour before it
erodes reasoning:

- A direct IQ2_XS-vs-IQ3_XXS comparison at 256K on one card reports IQ2_XS's
  work performance "stumbling", fine for summaries and chat but showing
  "semantic decay or hallucination" on long-chain reasoning, multi-condition
  boundaries, and precise format output such as hard JSON. It recommends IQ3_XXS
  outright: <!-- markdownlint-disable-line MD013 -->
  <https://lcz.me/topic/2192>
- A 21-task agentic benchmark across IQ4_XS → IQ3_XXS → IQ1_S scored 17 → 14 →
  11, and found the looser tier emitting tool calls in the **wrong XML dialect
  inside its thinking block** — invisible to the harness — plus failing to stop
  (27k–40k tokens, no output). <https://geekoyaji.com/tech/qwen38-flash-next-quantization-evo-x2-verdict/>
- A dual-3060 Strata test across all four tiers: smaller quants are "noticeably
  worse at multi-step generation tasks". <https://www.mindstudio.ai/blog/qwen-3-8-flash-next-strata-1000-build>

That matters because the intended use includes **execution-only coding
delegation**, where format adherence and stopping are the failure modes that
break a harness. The cost is real but bounded: the same direct comparison
measured IQ3_XXS at ~15% slower short-context decode, ~10% slower at 35K+, and
**1% faster prefill**, with MTP draft acceptance slightly *better*.

On the speed question the published R9700 figures are the wrong anchor. A single
R9700 measured **98.4 tok/s decode** on the IQ3 tier at 256K — but on a Ryzen 7
9700X over PCIe 5.0. This box sustains **~28.6 tok/s decode**, and the gap is
CPU-side expert compute, not the link (see
[Measured on this box](#measured-on-this-box)); a 60–90 tok/s expectation, which
this runbook carried until the first deployment, was never reachable on Zen 1.

Two tiers to know about but not use here: **IQ3_S** is the strongest quality
point overall, but at `STRATA_REF=v0.1.41` Swift has no IQ3_S — the setup table
restricts it to `families: ("qwen",)`, and Swift gained an IQ3_S tier in a later
engine ref. **Q2_0** is the fastest tier and had the best prompt throughput in
the ISTA-DASLab measurements, but its Q2_0 profile is likewise restricted to the
base model at this ref, and it trades away the most quality.

### The fine-tune

**Swift 1.5** is used rather than the base model: 63.4% fewer thinking tokens
and ~1.8x sooner answers at under 1% accuracy loss, which suits execution-only
delegation. Note its own licence (Swift Open License 1.0) rather than
Apache-2.0; `STRATA_FAMILY=qwen` switches to the base model at the same tier if
that matters, and both share shard 2 and a vision encoder.

### Why Strata rather than llama.cpp

`research/llm-engine-rdna4-options.md` and `research/llm-model-shortlist-32gb.md`
both assumed weights must be VRAM-resident, so both recommended llama.cpp and
both **excluded** Flash-Next ("weights alone are ~47 GB at IQ3"). Strata's
RAM/SSD expert offload invalidates that premise, which only became viable here
with 128 GB of RAM and an Optane. The llama.cpp research stands as the fallback
path if Strata disappoints; `research/radiance-vs-strata-rdna4.md` is the
newer single-card engine survey and reaches the same "stay" verdict.

## 1. Datasets

Both run on the TrueNAS host. Nothing needs a chown: the Strata image runs as
root, as the ComfyUI image does.

### The 900P link is verified — do not re-litigate it

Measured on this box (2026-10-13), the card at BDF `26:00.0`, `8086:2700`:

```
LnkCap: Port #0, Speed 8GT/s, Width x4, ASPM L0s, Exit Latency L0s <4us
LnkSta: Speed 8GT/s, Width x4
LnkSta2: Current De-emphasis Level: -3.5dB, EqualizationComplete+ EqualizationPhase1+
```

**PCIe 3.0 at full x4, at the card's own capability** — the best case, so the
900P's ~2.5 GB/s sequential rating is fully reachable rather than capped near
2 GB/s. `EqualizationComplete+` confirms it trained at Gen3 rather than falling
back. The drive sits on **NUMA node 1**, which is worth knowing when reading
benchmarks later, not worth changing.

Two notes for anyone re-checking this:

- `lspci -vv` needs root. Without it the device lists fine but reads
  `Capabilities: <access denied>`, and the `LnkCap`/`LnkSta` lines are simply
  absent — which can look like a missing card.
- `lspci -d 8086:2700` matches on the exact id, so a wrong id returns nothing.
  Find the device first with `lspci -nn | grep -iE "non-volatile|0108"`, or read
  the negotiated link unprivileged straight from sysfs:
  `cat /sys/bus/pci/devices/0000:26:00.0/current_link_{speed,width}`

### Fast pool — the model shards

```sh
zpool create -o ashift=12 -m /mnt/cistern cistern /dev/disk/by-id/<optane-900p-id>
zfs create -p -o recordsize=1m -o atime=off cistern/apps/strata/models
```

**`-m /mnt/cistern` is required, not cosmetic.** TrueNAS's root filesystem is
read-only, so a pool created without it gets `mountpoint=/cistern` and fails to
mount with `cannot mount '/cistern': failed to create mountpoint: Read-only
file system`. The pool is still created and healthy — only the mount fails — so
this is repairable in place rather than needing a destroy and recreate:

```sh
zfs set mountpoint=/mnt/cistern cistern
```

The UI does this for you when it creates a pool, which is why an existing pool
like `puddle` shows `ALTROOT /mnt` in `zpool list` and needs no such setting.

Confirm the id first with `ls /dev/disk/by-id/ | grep -i intel` — do not guess
it, and use the bare `nvme-INTEL_...` entry, never a `-part1` or `_1` alias.
`zpool labelclear` reporting `failed to clear label` on a blank drive is
expected, not an error: there is no label to clear. Single disk, no redundancy,
on purpose: everything under it is re-downloadable from Hugging Face at a
pinned revision. `recordsize=1m` suits the large sequential shard reads;
`atime=off` avoids write amplification on a read-mostly dataset.

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
`-00002-of-00002.gguf` file is a 38.8 GB n-gram/PLE lookup table that Strata
reads from disk *during inference* rather than loading into RAM. Everything
else in `/data` is read once at startup and lives in RAM afterwards — which is
why the packs and the MTP layer stay on the spinning pool. If this path ends up
on a rotational disk instead, setup.py detects it and either warns that random
n-gram reads "can stall prompts for minutes", or moves the whole table into RAM
with `--ple-io ram`.

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
| `cistern` | model shards + mmproj | ~71 GiB of ~250 GiB usable (~28%) |
| `puddle` | pack + mtp + config | ~50 GiB |

A 280 GB Optane is 260.8 GiB raw and about **250 GiB usable** after ZFS slop
space and metadata, so ~71 GiB of model files sits at **~28% occupancy**.
OpenZFS's actual hard guidance is only to keep free space above 10% — below 4%
free the metaslabs flip from first-fit to best-fit allocation and IOPS collapse.
The 80% figure is community convention, not a ZFS requirement. With one model
instead of two there is roughly 170 GiB spare, which is what makes the IQ2_XS
hedge in section 9 affordable if you ever want it.

## 2. Pre-seed the models

Do this **before** the first Deploy, to avoid a ~76 GB download inside a
container. `--gguf-dir` is not needed: shards hand-placed with their **original
names** in the expected subdirectory are validated against their own tensor
directory and marked whole, so setup skips the download entirely.

### The directory name is case-sensitive — get it from the log

`setup.py` builds the path as `models_dir / tag`, where the tag is the family
prefix plus the model name **uppercased**. The model directory therefore uses
the **mixed-case** tag while the config and pack use `tag.lower()`:

| Artefact | Path |
|---|---|
| Model shards | `models/`**`swift-IQ3_XXS`** (uppercase model) |
| Config | `config/`**`strata-swift-iq3_xxs`**`.json` (lowercase) |
| Pack | `packs/`**`swift-iq3_xxs`** (lowercase) |

Getting this wrong is silent in the worst way: setup creates the directory it
wants, finds the shards absent, and starts a fresh download beside your
pre-seed. **The authoritative answer is the first start's own log line** —
`The model files go in /data/models/<TAG>` — where `<TAG>` maps to
`$STRATA_MODELS_PATH/<TAG>` on the host. Confirm it before transferring 76 GB,
or check the layout against the table below.

To recover from a mismatch, stop the container and rename the directory to the
uppercase tag. **No finish mark is needed**: setup validates a hand-placed shard
against its own tensor directory and writes the mark itself (`#173`), so a
correctly named, complete file is accepted as-is.

Stop the container *before* renaming — it will be actively downloading into the
directory it created:

```sh
docker stop strata && docker rm strata

cd /mnt/cistern/apps/strata/models
ls -la                              # see which directory setup made and where your files are
rm -rf swift-IQ3_XXS                # the one setup created; confirm it holds no partial you want
mv swift-iq3_xxs swift-IQ3_XXS
ls -l swift-IQ3_XXS                 # sizes must be 44.90 / 38.84 GB, and 0.91 GB for the mmproj
```

Restart, and the log should skip the download and go straight to the pack step.

The revision pin matters. `setup.py` reads the Hub at pinned commits, and
`main` moves under it. Downloading from `main` risks a layout setup will not
recognise.

```sh
MODELS=/mnt/cistern/apps/strata/models
SWIFT_REV=99bb8f7f95c7aa7b24a36a7786a4f657b30f5d3d

mkdir -p "$MODELS/swift-IQ3_XXS"

# Swift 1.5 IQ3_XXS - 75.97 GB across two shards, plus its OWN vision encoder
# (a different file from the base model's).
curl -fL -C - --retry 3 -o "$MODELS/swift-IQ3_XXS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf"
curl -fL -C - --retry 3 -o "$MODELS/swift-IQ3_XXS/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf"
curl -fL -C - --retry 3 -o "$MODELS/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf"
```

Neither repo is gated, so no Hugging Face token is required.

**On `-C -`:** Hugging Face advertises `accept-ranges: bytes` on these
`resolve/<rev>/` URLs, so this resumes a partial download instead of starting
over. Verify it on the first large file rather than trusting it — the header
should read `HTTP/2 206` and curl should print a progress meter:

```sh
curl -sIL -H "Range: bytes=0-1" \
  "https://huggingface.co/ukisai/Swift-1.5-Qwen3.8-Flash-Next-GSQ-RCO-GGUF/resolve/$SWIFT_REV/Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf" \
  | grep -iE "^HTTP|content-range|accept-ranges"
```

Expected layout:

```
/mnt/cistern/apps/strata/models/
├── mmproj-Swift-Qwen3.8-Flash-Next-BF16.gguf                    0.91 GB
└── swift-IQ3_XXS/
    ├── Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf  44.90 GB
    └── Swift-Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf  38.84 GB
```

Sizes are the Hub's LFS metadata for this revision, so they double as a
truncation check. The directory name matches what `--gguf-dir` would expect, so
if a hand-placed shard is rejected you can point setup at it explicitly.

That is the whole pre-seed: two shards plus the vision encoder at the models
root. Everything else in `/data` is derived on first start:

| Artefact | Why it is not part of the pre-seed |
|---|---|
| **MTP draft layer** (~5 GB) | The only one of the three that *is* a network fetch, but not one you can `curl`. `mtp_fetch.py` pulls 31 tensors as byte ranges out of `Qwen/Qwen3.8-Flash-Next`'s 360 GB BF16 checkpoint, then `mtp_pack.py` and `mtp_rt.py` convert them into the runtime form the engine actually loads. **Those runtime files are not published anywhere.** Pre-seeding is possible but means running Strata's own three-stage pipeline outside the container — a host Python environment or a throwaway container, the right pinned revision, the `q2_0` expert quantisation, and a bind mount at `/data/mtp`. That is real work to remove 5 GB from the critical path, which is why it is deferred rather than impossible. It is the **last network activity** in the flow, in step 6. |
| **Pack** (`packs/swift-iq3_xxs`) | Derived from the shards by `tools/iq_pack.py`. For a native IQ pack this is seconds and needs no network at all — it indexes tensors and leaves the experts in the shard. |
| **n-gram/PLE table** (shard 2, 38.84 GB) | *Already covered* — it is the second shard you pre-seed, and the engine reads it from the Optane during inference. Nothing to fetch. |

**Expect the network to keep moving after the log says `model files present`.**
That is step 6's MTP fetch, and it is the only thing left that talks to the
internet. Setup has no `--download-only` flag.

If that fetch is interrupted, the recovery is **per tensor, not per byte**: each
tensor is digest-checked against the pinned revision and kept once it passes, so
a re-run re-fetches only what is missing. There is no `.part` file here —
`mtp_fetch.py` uses `Range: bytes=<start>-<end>` with 4 attempts and exponential
backoff per tensor, so a partially-read tensor restarts from its beginning
(3.4 GB worst case, not the whole layer). A flaky link therefore costs time, not
correctness. On a link that resets long transfers, `HF_ENDPOINT` (for example
`https://hf-mirror.com`) is honoured by this tool and the SHA-256 checks still
apply.

## 3. Deploy

1. **Add the hostname** (manual, per `apps/README.md`): in TrueNAS
   **Network → DHCP and DNS → Hostnames**, add `strata.absurdlab.dev` pointing
   at the Caddy box LAN IP, then refresh Passwall's DNS with the SIGHUP snippet
   in `apps/README.md` — do not hardcode the PID. Remove the old
   `strata-daily.absurdlab.dev` and `strata-coding.absurdlab.dev` entries if
   they are already there.
2. **Deploy `strata`** in Dockhand (stack name `strata`, compose path
   `truenas/apps/strata/compose.yaml`), with every key from `.env.example` in
   the env-var panel and **every key-icon toggle OFF**. Dockhand builds the
   image (a HIP engine compile, 10–20 minutes; `-j` is half the core count, so
   ~32 jobs on the 7551P), then starts the model. The first start also builds
   the pack and fetches the MTP layer, so allow well past the 600 s health
   start period before treating a slow start as a failure.

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
docker exec strata /opt/strata/engine/strata-device --list-devices

# Both mounts landed where intended
docker exec strata sh -c 'ls /data; ls /data/models; ls /data/models/swift-IQ3_XXS'

# The engine reused the baked build rather than recompiling
docker logs strata 2>&1 | grep -i "already built\|compiled"

# Which hipBLASLt tuning table setup applied (want gfx1201 / 100202 on 7.2.4)
docker logs strata 2>&1 | grep -i hipblaslt

# The PCIe probe and the expert-cache size it settled on: the two numbers that
# decide your throughput. Read these before tuning anything.
docker logs strata 2>&1 | grep -iE "pcie probe|expert cache|slots|kv streaming|kv-resident"

# The matrix-core prompt attention actually engaged (expect a "on" banner)
docker logs strata 2>&1 | grep -i "STRATA_HIP_WMMA\|matrix cores"
```

The health check is the image's own (`/health`, 600 s start period), so
`docker ps` shows `healthy` once the model is loaded, and `GET /v1/status`
reports what is running.

### Reaching it by hostname, and why the API key is a separate question

The direct LAN address works immediately; the Caddy hostname does **not**, and
fails with:

```json
{"error": {"type": "forbidden", "message": "Host 'strata.absurdlab.dev' is not allowed (DNS rebinding protection). ..."}}
```

That is not a proxy fault and not an access-control failure. `server.py` refuses
any request whose `Host` header it does not recognise, unless an API key is set,
and an IP address is not one of the names it knows — which is exactly why the
direct port works and the name does not. Fix it in the compose environment, not
the config file:

```
STRATA_ALLOWED_HOSTS=strata.absurdlab.dev
```

A bare name, a leading-dot suffix (`.absurdlab.dev` covers every subdomain) or
`*` are all accepted, comma-separated. Configuration rather than code is
deliberate: `setup.py` regenerates the config when a setting it presets changes,
so a value that lives only in `strata-swift-iq3_xxs.json` can be silently lost.

**The error message offers three fixes and they are not equivalent.** The key
distinction, from the source:

```python
def _authorized(self) -> bool:
    if not svc.api_key:
        return True          # no key configured: everything is authorized
```

```python
if svc.api_key or host_allowed(host, svc.host_names, "*" in svc.allowed_hosts):
```

So `allowed_hosts` **only inspects a header**, which any deliberate caller sets
freely — it exists to stop a browser on some other site being rebound to this
server, and it protects nothing against `curl`. The API key is the actual access
control, and it gates more than `/v1`: every control POST, `/config`,
`/settings`, `/load`, `/unload`, `/slots` and `/metrics`. Only genuinely static
assets (`/fonts/*`, `/web/*`) bypass it.

Note the second line: **the Host check runs only when no key is set.** Adding a
key turns it off entirely and the key becomes the whole check, so a key is not
additive with `allowed_hosts` — it replaces it.

This stack therefore ships without a key, and network placement (LAN-bound,
reached over the proxy network) is the control. If that stops being sufficient,
adding one is two changes, not one:

1. `API_KEY=<key>` as a **plain** env var in the Dockhand panel — never the
   key-icon secret toggle, since this stack is `restart: unless-stopped` and
   secrets do not survive a daemon-initiated restart.
2. An injection label on the service, or the web UI breaks — the page is served
   through Caddy and its own `/v1` calls would start returning 401:

   ```yaml
   caddy.header_up: "Authorization \"Bearer <key>\""
   ```

   The route labels carry no `header_up` today, so nothing injects the header
   for you.

The key is useful to any client that can be configured with it, Caddy or not —
the label exists only because the UI shares an origin with the API.

## 5. Benchmark it — your own numbers, first

Every published R9700 figure was measured on a PCIe 4.0/5.0 host. This box is
**PCIe 3.0 x16**, and the published CPU is much faster than a 7551P. The first
deployment has since put numbers on it ([Measured on this box](#measured-on-this-box)),
but re-measure whenever the quant tier, the context or the engine ref changes.

The engine reports per-request numbers with no tooling, but **not on container
stdout by default**. The format is

```
prompt P tokens = C reused + N read in R ms (X tok/s), G generated in M ms (Y tok/s)
```

where `X` is prefill (prompt read), `Y` is decode, and `C` is how much of the
prompt came from the prefix cache. Grep the engine's own log for it:

```sh
sudo grep -E "prompt [0-9]+ tokens" /mnt/puddle/apps/strata/strata-swift-iq3_xxs.log | tail -5
```

The friendlier re-statement — `request prompt P cached C output O prompt_read R ms
total S ms prefill X tok/s decode Y tok/s` — is printed to container stdout
**only when `STRATA_REQUEST_LINES=1`** is set (`serve/server.py`, `echo_requests`).
Without that variable, `docker logs … | grep "tok/s decode"` matches nothing;
that is expected, not a sign the engine is silent. What container stdout always
carries is the generation progress line:

```sh
docker logs strata 2>&1 | grep -E "answering:|done:" | tail -5
# [strata] answering: 3221 of max 261499 tokens, 28.4 tok/s, 118 s
# [strata] done: 3280 tokens in 119 s (28.6 tok/s) ..., expert cache 98.8% hit (+0.1% over PCIe)
```

That rate covers generated tokens over the request window, so on a long answer
after a short prompt it is close to pure decode; on a short answer after a long
prompt it is dragged down by prefill. Use the engine log's split line when the
two need separating.

Record `X` and `Y` at ~1K and again at ~32K prompt tokens — prefill is where a
256K window costs you, decode is flat in context. The published comparison
points, all on one R9700 at 256K with KV streaming, are **98.4 tok/s decode /
255 tok/s prefill** on a Ryzen 7 9700X over PCIe 5.0 and **87.6 tok/s decode at
131K**; the same page records **45.5–48.3 tok/s decode** for a Coder IQ1_M at
32K, which is the more honest shape of reference for this CPU class.

Then A/B the tuning in section 7 against that baseline, one change at a time —
and note that the first deployment already retired one of those candidates
(section 6): at a 98.8% expert-cache hit, `--pcie-frac` has almost nothing left
to move.

## 6. Measured on this box

First deployment, 2026-10-13: Swift 1.5 IQ3_XXS at 262144 context, one R9700,
EPYC 7551P, PCIe 3.0 x16, `STRATA_REF=v0.1.41`. Verbatim from the container log:

```
--max-context 262144 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 700; server
[strata] filling the GPU's expert cache (15271 experts, 24.79 GiB of VRAM) ...
[strata] done: 3280 tokens in 119 s (28.6 tok/s) (stop, cancel=False), expert cache 98.8% hit (+0.1% of the routed experts over PCIe)
```

| | Measured | Projected before the deploy |
|---|---|---|
| Expert slots resident | **15,271 / 24,576 = 62%** | ~10,000 (41%) |
| Expert cache in VRAM | 24.79 GiB | — |
| Cache hit rate | **96.2% → 98.8%** as the session warmed | — |
| Routed experts over PCIe | **0.1–0.5%** | assumed significant |
| Sustained decode | **28.6 tok/s** | 60–90 tok/s |
| Short-request rates | 252 tok in 35 s (7.9 tok/s) at 96.2%; 134 tok in 10 s (23.3 tok/s) at 98.0% | — |

Two corrections follow from this, and both were errors in this runbook rather
than in the build.

**The expert-residency projection was wrong, and wrong pessimistically.** The
model-choice table estimated ~41% residency for IQ3_XXS, derived from Strata's
Coder reference figures, and concluded that PCIe 3.0 would be the binding
constraint on decode. The card actually holds 62% of the routed experts and
misses 0.1–0.5% of them at runtime. **The Gen3 link is nearly irrelevant to
decode**, not the bottleneck. The projection has been corrected in place rather
than quietly, and the remaining unmeasured tiers are now marked as projections.

**The 60–90 tok/s expectation was never reachable on this CPU.** It was anchored
on 98.4 tok/s measured on a Ryzen 7 9700X over PCIe 5.0 — and on the *larger*
IQ3_S tier, so the deficit is if anything understated for IQ3_XXS. With only
0.1% of routed experts crossing PCIe, each token is bounded by expert and
attention compute rather than by any transfer, and a Zen 1 core is roughly a
third the speed of that host. **~28.6 tok/s sustained is the expected figure for
this box**, not a symptom of misconfiguration.

The rate is steady rather than decaying — 28.0 → 28.6 tok/s monotonically across
119 seconds — which matters because the web UI's live rate is, in Strata's own
words, "a rate over a window, not a mean since the first token". The UI showed
bursts near 80 tok/s; under MTP speculative decoding a forward pass emits `1 +
accepted drafts`, so windows where drafts land read high while the average
settles lower. **There is no separate fast "answer phase"**: thinking and answer
tokens go through the same decode path. Only the sustained figure is comparable
across configurations.

The two short requests are not evidence of a slow box. At 134 and 252 generated
tokens, fixed per-request and MTP warm-up costs dominate the window; only the
3,280-token request is a usable decode measurement. If those phases need
separating, the engine log's split line is the instrument — see section 5.

**No worker-count tuning is available here.** `--pool-workers` defaults to `0`,
which the engine documents as "every physical core minus the host's"
(`src/program/generate.cpp`), so on the 32-core 7551P it already runs ~31. Setup
only overrides it for hybrid Intel P/E CPUs (`hybrid_pool_workers`, which returns
None unless efficiency cores outnumber performance cores), and the 7551P is
homogeneous. The one related tip is `two_socket_note`: on 2-socket Xeons, 17–18
single-socket workers beat 35 by ~20%, because remote-socket expert workers are
slower. That is **conditional on the box being dual-socket** — check with
`lscpu | grep -i socket` before acting on it, and note it is a print-only
advisory that setup never writes to the config. The H11SSL-i is single-socket, so
nothing to do.

**What this does not establish**, stated because the measurement is narrow:
prefill at 32K or beyond is unmeasured, since every request above was a short
prompt with a long answer, and prefill is the phase a 256K window actually taxes.
Decode is measured at one point in a 262144-token window rather than near its
ceiling. This is a single session of a handful of requests whose prompts are not
recorded, so it is not a benchmark — it is what one deployment happened to
report, and the cache figures in particular were still climbing as the session
warmed, which says nothing about how a different workload would settle. One model
tier, one engine ref.

## 7. Tuning

Two switches exist, one is on, one needs a measurement first.

**On by default — `STRATA_HIP_WMMA=1`.** The int8-KV prompt attention on RDNA4
matrix cores, measured on this exact card at this ref at **7.2–7.5x** the
portable kernel (R9700 4K prompts 1,784 → 2,427 tok/s; 16K 1,797 → 2,700). It
requires `KV=int8` and pairs with a 96-slot prompt expert ring instead of 384.
It is **not bitwise** — greedy text can diverge from about token 50 — but the
accuracy cost was measured as within noise. Set `STRATA_HIP_WMMA=0` to A/B it.
Any value, `0` included, silences the engine's startup hint.

**Unverified at this ref — `STRATA_PF_FUSED=1`.** The fused int8 prompt-expert
kernels advertise +15–20% prompt throughput on gfx1201, but at `v0.1.41` they
sit behind a CUDA compute-capability check (`dev_info().cc >= 80`) that may not
open on RDNA4, so the gain cannot be assumed. Test it and believe only the log:

```sh
# Set STRATA_PF_FUSED=1 in the Dockhand panel, redeploy, send one prompt, then:
docker logs strata 2>&1 | grep -i "fused"
```

A banner naming the fused kernels means it engaged; no banner means it silently
fell back to MMQ and the switch is a no-op here. The quality trade is small and
documented (first-token KL against the FP16 prompt path was 0.00340 for MMQ vs
0.00388 for fused over a pooled 24-prompt IQ3_XXS sample; top-1 agreed 24 of
24), so it is worth having — just not worth claiming before you see it.

**The one engine-specific number worth overriding: `--pcie-frac`.** This share
of each layer's missed experts is read by the GPU over PCIe while the CPU
computes the rest. v0.1.41 sets it from a startup probe scaled against a 20 GB/s
reference (`base * min(1, gbps/20)`), so on PCIe 3.0 x16 expect roughly `0.34`
from the `0.55` native-pack base. That scaling was calibrated on machines whose
link is the bottleneck; on this host the Zen 1 CPU is at least as slow, so the
engine's own auto-balance may be right and is not obviously wrong. It is an
`args` entry, not an env var, so it lives in the config:

```sh
# Stop the stack first, then edit the recorded args (the volume copy, not the
# link in /opt/strata) and start it again.
CFG=/mnt/puddle/apps/strata/config/strata-swift-iq3_xxs.json
grep -o '"--pcie-frac", "[^"]*"' "$CFG"   # if present
```

Add `"--pcie-frac", "0.34"` to that args array, restart, and compare against
`0.25` and `0.45`. Keep whichever wins on your box and record it in the config
comment. Note the volume copy is the source of truth on later starts — editing
`/opt/strata` alone does nothing, and a redeploy that changes a setting the
launcher presets can regenerate the file.

**The measurement has since shrunk this lever's ceiling.** With a 98.8% cache
hit, only 0.1% of routed experts cross the link at all (section 6), so the share
the GPU reads over PCIe is already tiny and re-balancing it can move very little.
At roughly 2.6 MB per expert, even a 10% miss rate would add on the order of
0.2 ms per token against a ~35 ms budget, so this is not where the 28.6 tok/s
goes. Try it only if a future workload drives the hit rate down; otherwise leave
the probe's value alone and do not expect a gain.

**Do not** trade throughput for reproducibility. The four flags that make greedy
output byte-identical (`--prompt-cache 0 --adapt-swaps 0 --pcie-frac 0`,
`STRATA_IQ_MT_MIN=1`) cost decode and prefix reuse, and no harness needs them.

## 8. Day-2

- **Before starting ComfyUI:** stop `strata` in Dockhand. Do not overlap them.
- **Switch quant tier:** pre-seed the other tier's shards (section 2) and change
  `STRATA_MODEL`, then `REINSTALL=1`. Switching to a model already on the volume
  needs no reinstall; changing settings for an already-configured model does.
- **Toggle vision:** change `STRATA_VISION` and `REINSTALL=1`. `no` removes
  image input entirely and buys back the 700 MiB VRAM reserve (~2% decode at
  ~1.44 MiB/slot); `cpu` keeps it. `cpu` is not "make the CPU do the vision" as
  opposed to some other mode — it is the only vision mode on this card.
- **Bump the engine:** change `STRATA_REF` in the env panel, then Deploy with
  "build images" enabled. Faster starts, sharper Vision, and the RDNA4 prompt
  defaults of 0.1.43 are the reasons to consider it — see section 9.
- **Changing context or KV needs `REINSTALL=1`**, and each change re-plans the
  expert cache, so re-measure afterwards rather than assuming the old number.

## 9. Outstanding follow-ups

- **Measure prefill at depth.** Section 6 records decode at 28.6 tok/s but every
  request there was a short prompt with a long answer, so **prefill at 32K and
  beyond is still unmeasured** — and prefill is the phase a 256K window actually
  taxes. This is the largest remaining unknown in the runbook, and the engine
  log's split line (section 5) is the instrument for it.
- **Consider `STRATA_REF=v0.1.43` once you have a baseline.** It turns the
  RDNA4 prompt-path defaults on by itself — route tail skip, the measured PCIe
  share, 16K prompt chunks, and the fused IQ3 prompt experts that section 7
  cannot confirm at 0.1.41. That is the single largest remaining prompt-side
  gain, but it is a new engine to re-verify, so take it as a second step rather
  than part of the first deploy.
- **IQ2_XS hedge.** The Optane has ~170 GiB spare, so the 68 GB IQ2_XS tier can
  be pre-seeded alongside and swapped by `STRATA_MODEL` if IQ3_XXS proves too
  slow in practice. Keep both out of the same `packs/` name — the pack is keyed
  by `swift-iq3_xxs` / `swift-iq2_xs`, so they do not collide.
- **Vision encoder detail.** The CPU path caps a picture at 300 tokens
  (`max_tokens: 300`) where the GPU path allows 1024. For OCR that is the
  setting to raise first if small text reads badly: add `"min_tokens"` /
  raise the cap in the `"vision"` block of `strata-swift-iq3_xxs.json`. A Q8_0
  mmproj (590 MiB vs 865 MiB, cosine 0.997–0.999) is recommended by Strata's
  docs for `--vision cpu` but is not hosted in either HF repo; it must be built
  with llama.cpp. Deferred: BF16 works, and this is a size optimisation.
- **UniFi integration is no longer planned.** Camera-stream analysis belongs to
  the UniFi AI Key. The agent's vision stays for OCR and NAS video-vault
  indexing, which is why no `min_tokens` tuning for small-object grounding is
  needed here.
- **Supersession note still unwritten.** `research/llm-engine-rdna4-options.md`
  and `research/llm-model-shortlist-32gb.md` both excluded Flash-Next on the
  assumption weights must be VRAM-resident, and both are still untracked. They
  deserve a note explaining why Flash-Next is in scope; the reasoning currently
  lives only in this file.
