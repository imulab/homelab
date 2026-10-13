# Hermes vs OpenClaw for a self-hosted personal agent on the Strata endpoint

> Compiled 2026-10-13 via direct read of primary sources (official docs + GitHub
> API + the Strata source at `v0.1.41`). The background `/research` subagent
> stalled; this note was written by the driving session from the same sources.

## TL;DR

**Both harnesses clear the baseline, and the decision turns on feature set.** Each
is MIT-licensed, very large, actively maintained (pushed 2026-10-13), deployable
as a plain Docker Compose gateway (the Dockhand stack shape), and each can point
at the Strata endpoint (native tool calling — see
[research/strata-endpoint-api-surface.md](./strata-endpoint-api-surface.md)).

**Recommendation: Hermes (`NousResearch/hermes-agent`), on feature set.** Against
the operator's requirements — email threads as the primary interface, voice +
smart-home later, life-admin + occasional coding execution, multi-user — Hermes
wins because it is the only one of the two with a **native Email channel**
(threads + attachments + allowlists + pairing) and a **native Home Assistant
integration**. Hermes's messaging gateway covers 30+ platforms and runs as an
official s6-supervised Docker image. OpenClaw is the pick only if the interface
shifts to its chat-channel/voice-wake strengths and email + smart-home drop out.

## The candidates

| | OpenClaw (canonical) | Hermes Agent |
|---|---|---|
| Repo | [openclaw/openclaw](https://github.com/openclaw/openclaw) | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| License | MIT | MIT |
| Language | TypeScript / Node | Python |
| Stars (API, 2026-10-13) | 391,853 | 253,348 |
| Last push | 2026-10-13 (active) | 2026-10-13 (active) |
| Homepage | https://openclaw.ai | https://hermes-agent.nousresearch.com |
| Tagline | "The AI that really does things. Any OS. Any Platform." | "The agent that grows with you" |

## "A distribution of OpenClaw" — which one to use

The canonical project is **`openclaw/openclaw`** (homepage openclaw.ai, 392k
stars, active). Two things orbit it and are easy to confuse:

- **`jakekwak/openclaw`** — a Korean-language fork (`docs.openclaw.kr`), 0 stars,
  last pushed 2026-03-23. **Do not use**; it is a stale fork of the canonical
  repo, not a maintained distribution.
- **"Claw" (`claw.so`, ClawHQ)** — a related desktop-app ecosystem whose docs
  (`docs.claw.so/engine/…`) document the same **OpenClaw Engine** (its "OpenClaw
  Setup" and "OpenClaw Engine" sections are the OpenClaw gateway). The docs are a
  useful reference but the runnable gateway is the canonical `openclaw/openclaw`.

Net: **there is no reason to pick a fork.** Use the canonical `openclaw/openclaw`.

## Docker / compose deployability (the hard requirement)

- **Hermes — first-class.** Official image `nousresearch/hermes-agent` (tags
  `latest`/`stable`, `main`, `X.Y.Z`, pin a digest), amd64 + arm64, Python 3.14.
  The gateway runs supervised by `s6-overlay`; one `/opt/data` volume holds all
  state (`.env`, `config.yaml`, `state.db` SQLite, sessions, memories, skills).
  The docs give a verbatim `docker-compose.yaml` (gateway on `8642`, optional
  dashboard on `9119`) with `restart: unless-stopped`. A documented caveat:
  SQLite `state.db` uses WAL, which a virtiofs/9p bind mount can corrupt — set
  `database.journal_mode: delete` on non-native mounts (ZFS bind mount on
  TrueNAS is a native mount, but worth setting explicitly).
- **OpenClaw — supported, slightly more installer-shaped.** Pre-built gateway
  image at `ghcr.io/openclaw/openclaw` (tags `main`, `latest`, `<version>`),
  plus a `./scripts/docker/setup.sh` that builds/onboards and starts via Docker
  Compose. Control UI on `http://127.0.0.1:18789/`. Needs ≥2 GB RAM for a local
  build (a non-issue on this 128 GB box). Docker is labelled "optional" — the
  project is gateway-first — but the containerized gateway is a documented,
  supported path.

Both map onto the `truenas/apps/<name>/compose.yaml` convention. Hermes's path is
more literal (official image + compose example); OpenClaw's needs one extra step
(pin the GHCR image and write the compose ourselves, since its script does both).

## Local / custom model support

- **Hermes:** a first-class **"Custom endpoint"** provider (saved in
  `config.yaml`), plus `model.base_url` for local daemons and dedicated
  Ollama / vLLM / LM Studio providers. Prior repo research records that Hermes
  **rejects local servers with <64k effective context** and supports
  `model.supports_vision` for custom-endpoint vision. Strata's 256K context and
  OpenAI tool calling satisfy both.
- **OpenClaw:** providers include Ollama, OpenAI, Qwen, GLM, MiniMax, etc., and
  a dedicated "Local Models" gateway page covering LM Studio and **"other
  OpenAI-compatible local proxies."** So a custom OpenAI-compatible base URL
  (Strata) is a documented pattern, though the exact config key is less centrally
  documented than Hermes's "Custom endpoint" wizard.

## Tool calling against a local model

This was the load-bearing question and it is resolved **in Strata's favour**:
the endpoint natively implements `tools`/`tool_choice` and returns first-class
`tool_calls` (streamed and non-streamed), speaking OpenAI
`/v1/chat/completions`, Anthropic `/v1/messages`, and OpenAI `/v1/responses`.
The model's tool dialect is Qwen-native XML, which the server parses — so a
harness receives real function calls, not text to scrape. Neither harness needs
anything vLLM/Claude-specific here.

The remaining risk is **latency, not correctness**: Strata serves one sequence at
a time (~28.6 tok/s decode), and an agentic turn issues many model calls. A heavy
harness will feel slow regardless of which one is chosen; this is a property of
the endpoint, not the harness pick.

## Channels / UI

- **Hermes:** a single messaging gateway covering **30+ platforms** — Telegram,
  Discord, Slack, WhatsApp, Signal, SMS, **Email**, **Home Assistant (plugin)**,
  Mattermost, Matrix, DingTalk, Feishu/Lark, WeCom, Weixin, BlueBubbles
  (iMessage), Photon (iMessage), QQ, Yuanbao, Microsoft Teams, LINE, ntfy, and
  more — plus a CLI/TUI, desktop app, web dashboard, and Voice Mode (TTS replies,
  voice-note transcription, Discord voice channels). Email supports images,
  files, and threads; a dedicated `openmail` plugin and `EMAIL_ALLOWED_USERS` /
  email pairing sit on top.
- **OpenClaw:** ~20 chat channels (Telegram, WhatsApp, Signal, iMessage, Discord,
  Slack, Google Chat, LINE, Matrix, Mattermost, MS Teams, Nextcloud Talk, Nostr,
  Twitch, Zalo, BlueBubbles, grammY) plus a Control UI / WebChat / TUI and voice
  call/wake features. **Gmail appears only as an automation trigger** ("Gmail
  PubSub"), not a bidirectional email conversation channel.

## Security / self-host posture

- **Hermes:** gateway + dashboard in one supervised container; the dashboard
  **fails closed without an auth provider on any non-loopback bind** (a June 2026
  hardening after an exposed-dashboard campaign). On the LAN behind Caddy this
  means configuring the bundled username/password provider or binding loopback.
  Secrets live in `/opt/data/.env`.
- **OpenClaw:** gateway token written to `.env`; sandboxing / tool-policy /
  "elevated mode" give per-tool blast-radius control; the Control UI is the ops
  surface. Channel-specific hardening (e.g. Telegram allowlists) is in progress
  per the docs' experiment plans.
- **Dockhand caveat applies to both:** any credential injected as a Dockhand env
  var does not survive a daemon-initiated restart (host reboot) and must be
  redeployed to re-inject. The harness's secrets should therefore live in its own
  persistent config/volume, not only in Dockhand's env-var panel.

## Comparison

| Axis | Hermes | OpenClaw (canonical) |
|---|---|---|
| License / maintenance | MIT, active, 253k stars | MIT, active, 392k stars |
| Docker compose gateway | Official image + s6 + compose example | GHCR image + setup script + compose |
| Local OpenAI-compatible endpoint | "Custom endpoint" + base_url, explicit | OpenAI-compatible proxies (documented) |
| Tool calling vs Strata | Works (native tool_calls) | Works (native tool_calls) |
| Channels | 30+ incl. Email, SMS, Home Assistant | ~20 chat platforms, no email channel |
| Data model | one `/opt/data` volume, profiles | gateway `.env` + config, Control UI |
| Docker docs quality | higher (dedicated, current) | solid but "optional"/installer-shaped |

## Recommendation

**Use Hermes (`nousresearch/hermes-agent`).** Evaluated on feature set (not
deployment ergonomics), it is the only candidate with a native **Email** channel
and a native **Home Assistant** integration, and it matches the operator's
requirements — email threads as the primary interface, voice + smart-home later,
life-admin plus occasional coding execution, multi-user (admin/user tiers,
allowlists, pairing). Point it at Strata via the "Custom endpoint" provider (the
≥64k-context expectation is satisfied by Strata's 256K). Deployment is one new
`truenas/apps/<name>/` stack joining the `proxy` network and calling Strata
container-to-container.

OpenClaw remains the pick only if the primary interface shifts to a chat channel
OpenClaw handles better (voice-wake nodes, WhatsApp/Signal/iMessage breadth) and
email/Home-Assistant drop out of the requirements.

## Sources

- [openclaw/openclaw](https://github.com/openclaw/openclaw) · [jakekwak/openclaw](https://github.com/jakekwak/openclaw) · [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — identity, license, stars, activity via the GitHub API (2026-10-13).
- [Hermes Docker Setup](https://hermes-agent.nousresearch.com/docs/user-guide/docker) — official image, s6 supervision, `/opt/data`, compose example, SQLite WAL caveat, dashboard auth gate.
- [Hermes LLM and Model Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers) — "Custom endpoint", `model.base_url`, Ollama/vLLM/LM Studio.
- [OpenClaw Docker (openclaw.kr)](https://docs.openclaw.kr/install/docker) · [OpenClaw/Claw Docker (claw.so)](https://docs.claw.so/engine/install/docker/) — GHCR image, setup script, Control UI, compose gateway.
- [OpenClaw Local Models](https://docs.claw.so/engine/gateway/local-models/) — LM Studio + "other OpenAI-compatible local proxies".
- [OpenClaw Model Providers](https://docs.claw.so/engine/providers/) — Ollama, OpenAI, Qwen, GLM, etc.
- Local: [research/strata-endpoint-api-surface.md](./strata-endpoint-api-surface.md) (Strata native tool calling), [research/llm-engine-rdna4-options.md](./llm-engine-rdna4-options.md) (Hermes ≥64k-context + `supports_vision`), [truenas/apps/strata/README.md](../truenas/apps/strata/README.md) (the deployed endpoint).
