# syntax=docker/dockerfile:1
#
# Strata (Niko1221/Strata) engine image for the AMD Radeon AI PRO R9700
# (RDNA4, gfx1201) over the HIP backend, for TrueNAS SCALE 26 (kernel 6.18).
#
# !! THIS FILE IS DUPLICATED - KEEP BOTH COPIES BYTE-IDENTICAL !!
# Each Strata stack carries its own copy, because Dockhand clones only the
# directory holding a stack's compose file, so a build context outside the
# stack directory is not reliably available:
#   truenas/apps/strata-daily/Dockerfile
#   truenas/apps/strata-coding/Dockerfile
# A stale copy silently produces a different image depending on which stack was
# deployed last. Both files being identical makes this the whole check:
#   diff -q truenas/apps/strata-daily/Dockerfile truenas/apps/strata-coding/Dockerfile
#
# The engine is compiled HERE, at image build time, so the container never
# compiles. setup.py decides whether to reuse an engine by reading
# engine/BUILD.json: it recompiles unless `backend == "hip"`, its `src` hash
# matches source_hash(ENGINE_SOURCES), and `archs` covers the card. Because
# build_engine_hip() writes that file itself, a container start logs
# "engine already built for this PC" and goes straight to serving.
#
# The Strata source is cloned at a pinned ref rather than vendored, so the
# Dockhand build context is just this directory - a few KB, no submodules.
#
# The build host's CPU matters: build_engine_hip() records cpu_floor() from
# /proc/cpuinfo into BUILD.json, and the runtime compares it. Dockhand builds
# on the NAS itself (EPYC 7551P, AVX2 => floor ""), which is the same CPU the
# container runs on, so the recorded floor matches. Building this image on a
# different machine (an AVX-512 host, say) records a floor that does not
# match, and the first container start recompiles the engine inline.
#
# ROCm version is load-bearing, not incidental. 7.2.4 ships hipBLASLt 1.2.2,
# which is exactly the version tools/hip/gfx1201-hipblaslt-100202.txt is
# calibrated for, so setup.py applies that tuning table at first start. The
# Strata docs measured the pairing on an R9700 with the full IQ3_XXS: a fresh
# 32K prompt read at 638 -> 1177 tok/s and a 7K one at 656 -> 1164, decode
# unchanged. Moving this base image to another ROCm line silently drops the
# table (setup falls back to plain hipBLAS: same answers, much slower prompts).
#
# Reference: https://github.com/Niko1221/Strata/blob/main/docs/AMD_HIP.md

FROM rocm/dev-ubuntu-24.04:7.2.4-complete

# Required: the Strata release tag or commit to build. No default on purpose -
# a silent "latest" is how a GPU stack drifts without anyone noticing. Set it
# in the stack's .env; see .env.example for the current value and how to check
# for a newer one.
ARG STRATA_REF

# gfx1201 is the R9700. Kept as an ARG so a future card is a one-line change.
ARG HIP_ARCH=gfx1201

# "cpu" builds the image encoder that runs on the CPU. The AMD backend has no
# GPU image encoder yet, so this is the only vision path on this card; "none"
# drops vision and shrinks the image.
ARG BUILD_VISION=cpu

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    LANG=C.UTF-8 \
    STRATA_EXECV=1

# cmake/ninja are what setup.cmake_build() shells out to; build-essential and
# git are what build_engine_hip() checks for before it will start.
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential ca-certificates curl git libatomic1 libgomp1 \
        python3 python3-pip python3-venv unzip cmake ninja-build \
    && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/Niko1221/Strata.git /opt/strata \
    && cd /opt/strata \
    && git checkout "${STRATA_REF}" \
    && git submodule update --init --recursive

WORKDIR /opt/strata

RUN python3 -m venv .venv \
    && .venv/bin/pip install --no-cache-dir --upgrade pip \
    && .venv/bin/pip install --no-cache-dir -r requirements.txt \
    && chmod +x setup.sh docker-entrypoint.sh

# Compile the HIP engine (and, when BUILD_VISION=cpu, the CPU image encoder)
# into engine/. This calls setup.py's own builder rather than driving cmake by
# hand, which is what guarantees BUILD.json is written in the exact shape the
# runtime freshness check expects. get_llama_cpp() fetches the llama.cpp
# revision this Strata ref pins, so the build needs network access.
RUN .venv/bin/python - <<'PYEOF'
import os, setup

arch = os.environ.get("HIP_ARCH", "gfx1201")
vision = os.environ.get("BUILD_VISION", "none")
llama = setup.get_llama_cpp()
setup.build_engine_hip({"arch": arch, "archs": [arch]}, llama, vision=vision)
PYEOF

# The cmake trees are build-time only; engine/ is what the container needs.
RUN rm -rf build-hip build-vision

VOLUME ["/data"]
EXPOSE 8080

# A long start period is deliberate: the first start downloads and prepares
# the model (tens of GB), and the port only opens once the model is loaded.
HEALTHCHECK --interval=30s --timeout=5s --start-period=600s --retries=3 \
  CMD curl -fs "http://127.0.0.1:${PORT:-8080}/health" || exit 1

ENTRYPOINT ["./docker-entrypoint.sh"]
