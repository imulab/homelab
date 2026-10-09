# strata-coding

The coding profile of the two **Strata** stacks. The full runbook — dataset
creation, model pre-seed downloads, deploy order, verification — lives in
[`../strata-daily/README.md`](../strata-daily/README.md). Read that first; this
stack shares its `/data` layout, model tree and MTP layer.

| | |
|---|---|
| Model | Qwen3.8-Flash-Next **Coder IQ1_M** |
| Context | 128K (`131072`) |
| KV | `k8v4` |
| Vision | `cpu` |
| RAM cap | 48 GB |
| Auto-start | **no** (`restart: "no"`) |

**Only one of `strata-coding`, `strata-daily` and `comfyui` may run at a time.**
There is no VRAM isolation between ROCm processes.

To use it: stop `strata-daily` in Dockhand, wait for it to exit cleanly, then
start `strata-coding`. `restart: "no"` means it does not come back by itself
after a reboot — that is deliberate, so the daily driver keeps the GPU when
you are not at the keyboard.

`Dockerfile` is duplicated from `../strata-daily/Dockerfile`; the two must stay
byte-identical (see the header note in either file).
