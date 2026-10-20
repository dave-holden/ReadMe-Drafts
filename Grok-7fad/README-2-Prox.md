<p align="center">
  <img src="../../public/assets/images/proxytool.webp" alt="proxytool" width="200">
</p>

# proxytool

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)
[![Python 3.10–3.14](https://img.shields.io/badge/python-3.10%E2%80%933.14-blue)](https://www.python.org)
[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)

I built this because I was tired of watching opencode and pi print JSON tool-call objects instead of actually running them.

**What it does in one sentence:** you run `proxytool serve`, point your agent at `http://127.0.0.1:8180/v1`, and the proxy takes care of the rest — routing, sanitising requests, repairing leaked tool calls (streaming and non-streaming), and stopping the model from looping forever.

Technically it’s a FastAPI proxy that sits between any OpenAI-compatible client (opencode, pi, curl…) and local inference backends (llama-server, mlx_lm.server, omlx, llama-swap, or anything else that speaks the same dialect). All the repair logic comes from the sibling library [`toolpatch`](../../packages/toolpatch/README.md).

When a model’s chat template is missing or mismatched it emits the tool call as **text inside** `choices[*].message.content` instead of the structured `tool_calls` field. Clients then just print the raw JSON. proxytool catches it in-flight and turns it into a proper tool call.

It also:

- Sanitises outbound `tools` / `tool_choice` (MLX doesn’t like `tool_choice`)
- Heals broken JSON (trailing commas, smart quotes, raw newlines, fences…)
- Normalises history so llama-server doesn’t 400 on multi-turn sessions
- Guarantees a `finish_reason` chunk so agents don’t re-prompt endlessly
- Reassembles split SSE frames safely
- Detects and aborts repetition loops (LoopGuard)

**Status:** v2.0.0 — GGUF and MLX both verified end-to-end with real agents.

> All repair logic lives in [`packages/toolpatch`](../../packages/toolpatch/README.md). Workspace overview: [root README](../../README.md).

## The rig

```
┌────────────┐ 8180 → ┌───────────────────┐ 8181 upstream → ┌────────────────┐
│ opencode / │        │ FastAPI proxy     │                 │ Qwen (GGUF/MLX)│
│ pi / curl  │───────►│ sanitize + repair │────────────────►│ or fake        │
└────────────┘◄───────┤ (this app)        │◄────────────────└────────────────┘
```

## The `proxytool` CLI

Thin launcher around the FastAPI app. Installing the package puts `proxytool` on your PATH.

```bash
uv sync --all-packages     # workspace
uv run proxytool serve     # default: 127.0.0.1:8180 → http://127.0.0.1:8181
uv run proxytool version
```

Once published: just `pip install proxytool` then `proxytool serve`.

### Subcommands

| Subcommand | Purpose |
|---|---|
| `serve` | run in the foreground |
| `start` | daemonise (waits for /health) |
| `stop` | graceful stop |
| `status` | running / stopped / stale |
| `health` | JSON probe of proxy + backends |
| `models` | what `/v1/models` returns from each upstream |
| `config` | diagnose the config file (`print` / `check`) |
| `version` | print version |

### Common flags (every flag has an env-var equivalent — flags win)

| Flag | Env var | Default | Purpose |
|---|---|---|---|
| `--host` | `PTP_HOST` | `127.0.0.1` | bind host |
| `--port` | `PTP_PORT` | `8180` | bind port |
| `--upstream NAME=URL` | `PTP_UPSTREAM_<NAME>` | — | register a backend |
| `--upstream-models NAME=m1,m2` | `PTP_UPSTREAM_<NAME>_MODELS` | — | model ids for that backend |
| `--default-backend NAME` | `PTP_DEFAULT_BACKEND` | first registered | fallback |
| `--upstream-url URL` | `PTP_UPSTREAM_URL` | `http://127.0.0.1:8181` | single-backend shortcut |
| `--model-map JSON` | `PTP_MODEL_MAP` | — | client alias → real model id |
| `--upstream-timeout S` | `PTP_UPSTREAM_TIMEOUT` | `300` | covers cold starts |
| `--loop-nudge N` / `--loop-abort N` | `PTP_LOOP_NUDGE` / `PTP_LOOP_ABORT` | `3` / `6` | LoopGuard (0 = off) |
| `--trace` | `PTP_TRACE` | off | dump payloads to stderr |
| `--log-level …` | `PTP_LOG_LEVEL` | `info` | log level |
| `--config PATH` | `PTP_CONFIG` | discovery | TOML config file |

### Upstream selection

```bash
# A. Single backend (most common)
proxytool serve --upstream-url http://127.0.0.1:8181

# B. Multi-backend routing by model id
proxytool serve \
  --upstream gguf=http://127.0.0.1:8181 --upstream-models gguf=qwen-gguf,qwen \
  --upstream mlx=http://127.0.0.1:8281  --upstream-models mlx=qwen-mlx \
  --default-backend mlx
```

proxytool never starts the model servers itself — it fronts whatever is already listening. For automatic start/stop use llama-swap (or compose/systemd).

## Quickstart (manual)

```bash
uv sync --all-packages

# Terminal 1 – fake or real upstream on 8181
uv run uvicorn fake_upstream:app --host 127.0.0.1 --port 8181
# or real:
# llama-server --host 127.0.0.1 --port 8181 --jinja --model $MPATH/qwen….gguf &

# Terminal 2 – the proxy
PTP_TRACE=1 uv run proxytool serve --upstream-url http://127.0.0.1:8181

# Terminal 3 – a client
curl -s http://127.0.0.1:8180/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen","stream":false,
       "messages":[{"role":"user","content":"list the files"}],
       "tools":[{"type":"function","function":{"name":"list_dir","parameters":{"type":"object","properties":{"path":{}}}}}]}' \
  | jq '.choices[0].message'
# → tool_calls populated; content clean
```

## Real backends

```bash
# GGUF
llama-server --host 127.0.0.1 --port 8181 \
  --model $MPATH/qwen2.5-coder-14b-instruct-q4_k_m.gguf \
  --ctx-size 16384 --jinja

# MLX
python -m mlx_lm.server --model mlx-community/Qwen2.5-Coder-14B-Instruct-4bit --port 8181
```

`--jinja` helps llama-server use the proper chat template. Everything the template still leaks (fenced/bare JSON, bad history shapes, etc.) is what this proxy fixes.

## End-to-end with llama-swap + agents

**1. llama-swap config** (spawns backends on demand):

```yaml
# ~/.llama-swap/config.yml
health-check: true
models:
  qwen-gguf:
    cmd: llama-server --host 127.0.0.1 --port ${PORT} --ctx-size 16384 --jinja --model /models/qwen2.5-coder-14b.gguf
    ttl: 300
  qwen-mlx:
    cmd: python -m mlx_lm.server --model mlx-community/Qwen2.5-Coder-14B-Instruct-4bit --port ${PORT}
    ttl: 300
```

**2. Start the stack:**

```bash
llama-swap --config ~/.llama-swap/config.yml      # :8080
proxytool serve --upstream-url http://127.0.0.1:8080 --upstream-timeout 600
# or daemonised:
# proxytool start --upstream-url http://127.0.0.1:8080 --upstream-timeout 600
```

**3. Point the clients at the proxy (never at llama-swap directly):**

```bash
export OPENCODE_CONFIG=./examples/opencode-gguf.jsonc
opencode

pi --provider proxytool --model qwen-mlx -p "list the files in ."
proxytool health && proxytool models
```

## opencode & pi configs

Ready-made examples live in [`examples/`](../../examples/). They all point `baseURL` at `http://127.0.0.1:8180/v1`. Aliases are rewritten server-side so the clients stay simple.

## Tests & checks

```bash
uv run pytest          # unit + regression + integration
uv run ruff check .
uv run pyright
```

Live / real-client tests are opt-in:

```bash
PTP_LIVE=1 uv run pytest apps/proxytool/tests/test_live_qwen.py -q
PTP_LIVE=1 uv run pytest apps/proxytool/tests/test_live_opencode.py -q
PTP_LIVE=1 uv run pytest apps/proxytool/tests/test_live_pi.py -q
```

They auto-skip unless the upstream is healthy and the client binary is on PATH.

## Config file

TOML discovery order and full schema are documented in the source and in `examples/proxytool/proxytool.toml`. `proxytool config print` and `proxytool config check` are your friends.
