# Strata endpoint API surface (native tool calling)

> Compiled 2026-10-13 by direct read of the Strata source at the deployed ref
> `v0.1.41` (`serve/server.py`, `serve/responses.py`, `serve/structured.py`,
> `serve/chat_template.jinja`). This is the endpoint the personal-agent harness
> will consume; it was not documented explicitly in the runbook
> (`truenas/apps/strata/README.md`), so it is recorded here as its own note.

## TL;DR

**Strata natively implements OpenAI-style function/tool calling.** The endpoint
accepts a `tools` array and `tool_choice`, returns structured `tool_calls` in
both streamed and non-streamed responses, and additionally speaks the Anthropic
`/v1/messages` dialect and the OpenAI `/v1/responses` dialect. This resolves the
load-bearing question for the agent-harness decision: **both Hermes and OpenClaw
can do real agentic tool use against this endpoint** — the harness does not have
to parse tool calls out of free text, because the server does the parsing and
returns first-class `tool_calls`.

## Endpoints

From the `serve/server.py` module docstring and route dispatch:

| Endpoint | Dialect | Notes |
|---|---|---|
| `POST /v1/chat/completions` | OpenAI | stream and non-stream; `tools`, `tool_choice`, `response_format` |
| `POST /v1/messages` | Anthropic | stream and non-stream; tools and `tool_choice` |
| `POST /v1/responses` | OpenAI Responses | stateless, built on the chat path (`#451`); function/namespace/custom tools |
| `GET /v1/models`, `GET /models` | — | model + capability listing |
| `GET /props`, `GET /slots`, `GET /health`, `GET /mcp` | — | status / control |

The `/v1/status` payload advertises `"dialects": ["/v1/chat/completions",
"/v1/messages", "/v1/responses"]` and `"structured_output": {"formats":
["json_object", "json_schema"], "method": "prompt_and_validate",
"constrained_decoding": false, "stream_buffered": true}`.

## Tool calling (function calling)

- The request `tools` array is accepted on all three dialects. On the Responses
  path, `request_tools(req)` reads `req.get("tools")` and maps each tool into the
  template form (`serve/responses.py:236-251`).
- `tool_choice` is honoured: `tool_choice_of(req.get("tool_choice"))`, including
  `none`, `any`, and a forced call to a named tool (a bad value is a 400 before
  anything is sent) (`serve/server.py:4944-4946`, `:3629`).
- Non-streamed chat responses emit `"tool_calls": [{"function": {"name": ...,
  "arguments": ...}} ...]` (`serve/server.py:3910`).
- Streamed responses emit chunked `tool_calls` deltas (`index`, `id`,
  `type: function`, then `function.arguments` fragments), merged back together by
  index on the client side, with `finish_reason: "tool_calls"`
  (`serve/server.py:3980-3994`, `:4024-4039`).
- The `OutputParser` runs with `stream_tools=True` and an opt-in
  `tool_call_recovery` mode (`serve/server.py:3415`, `:2657`).

## The model's tool dialect

The server does not require the model to speak OpenAI JSON. The chat template
injects tools as an XML `<tools>...</tools>` block and instructs the model to emit
`<tool_call><function=name><parameter=...>value</parameter></function></tool_call>`
XML; the server's parser converts that into OpenAI `tool_calls`
(`serve/chat_template.jinja:72-87` for the instruction, `:124-155` for rendering
assistant `tool_calls` back into the template). This is the Qwen native tool-call
format, so it plays to the deployed Qwen3.8-Flash-Next model rather than against
it.

## Structured output

`response_format` supports `json_object` and `json_schema`, implemented as
prompt-and-validate (not constrained decoding), stream-buffered
(`serve/server.py:3126`, `serve/structured.py`). Useful for agents that need a
final JSON answer.

## MCP

`serve/mcp.py` serves MCP tools, opt-in per request via `"strata_mcp": true`
(other clients "see exactly the API they always saw") — the web app uses it.
`GET /mcp` lists them.

## Concurrency caveat (matters for a shared endpoint)

The `serve/server.py` docstring states: **"One sequence at a time behind a FIFO
(plan: one resident sequence)."** So at `v0.1.41` the serving path is a single
serialized queue; a `#465` "parallel": N batch slots path exists and
`/v1/status` reports a `concurrency` block, but until that is verified on this
box, treat the personal agent, the family web UI, and any other client as
competing for one slot. Requests whose prompt + max tokens exceed the engine's
context are rejected with 400, never truncated (docstring).

## Sources

- `serve/server.py` at `v0.1.41` (module docstring: endpoints; `:3126`
  structured_output; `:3415` OutputParser; `:3910` non-stream tool_calls;
  `:3980-3994`, `:4024-4039` streamed tool_calls; `:4688` route dispatch;
  `:4944-4946` tool_choice; `:5846` ready banner).
- `serve/responses.py` at `v0.1.41` (`:236-251` `request_tools`).
- `serve/structured.py` at `v0.1.41`.
- `serve/chat_template.jinja` at `v0.1.41` (`:72-87`, `:124-155`).
- Repo: [github.com/Niko1221/Strata](https://github.com/Niko1221/Strata) (read at
  tag `v0.1.41` via the GitHub contents API).
