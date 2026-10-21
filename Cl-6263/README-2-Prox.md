![proxytool](public/assets/images/proxytool.webp)

# proxytool

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)
[![Python 3.10–3.14](https://img.shields.io/badge/python-3.10%E2%80%933.14-blue)](https://www.python.org)
[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)

**A small proxy that sits between your coding agent and your local model, and fixes tool calls the model fumbles.**

Local models sometimes write their tool call as plain text in the chat instead of the structured `tool_calls` field. Your agent can't run text, so you get raw JSON in the terminal instead of a directory listing. `proxytool` catches that on the way back, rebuilds a proper tool call, and your agent carries on none the wiser.

```
agent (opencode, pi, curl…)  ──▶  proxytool :8180  ──▶  your model server :8181
                             ◀──  (repairs tool calls)  ◀──
```

You point your agent at `http://127.0.0.1:8180/v1` and forget about it.

> **Heads up:** proxytool fronts servers that are *already running*. It doesn't start or stop `llama-server` or `mlx_lm.server` for you. If you want models loaded on demand, put [llama-swap](https://github.com/mostlygeek/llama-swap) (or a compose/systemd unit) behind it. proxytool does manage *itself*, though: `start`, `stop` and `status` are all there.

The repair logic itself lives in the sibling library, [`toolpatch`](../../packages/toolpatch/README.md). Workspace overview is in the [root README](../../README.md).

## What it does

- **Fixes leaked tool calls** in the model's reply: Hermes-style `<tool_call>` tags, fenced `` ```json `` blocks, or bare `{"name": …}` JSON sitting in prose. Works while streaming too, without making you wait for the first token.
- **Heals broken JSON**: trailing commas, raw newlines, smart quotes, stray fences.
- **Tidies requests going the other way**: validates `tools` / `tool_choice` (MLX doesn't support `tool_choice`, so it's dropped there) and merges the odd assistant-message shapes that make llama-server throw a `400`.
- **Leaves clean traffic alone**: if nothing's broken, what goes in comes out byte-for-byte.
- **Always finishes the stream**: the repaired stream ends with a `finish_reason` chunk before `[DONE]`, so agents close the turn instead of re-prompting.
- **Survives awkward chunking**: SSE lines are reassembled before decoding, so a `data:` payload or a multibyte character split across chunks isn't lost or mangled.
- **LoopGuard**: spots your agent sending the exact same request over and over, nudges the model, then pulls the plug after a few more.

## Install

```sh
uv tool install proxytool     # or: pip install proxytool
```

That puts a `proxytool` command on your PATH and pulls in `toolpatch` as a dependency.

Working from a clone? `uv sync --all-packages` and use `uv run proxytool …` instead.

## Quick start

**1. Start a model server** (any OpenAI-compatible one; this is llama.cpp on `:8181`):

```sh
llama-server --host 127.0.0.1 --port 8181 --jinja --ctx-size 16384 \
  --model "$MPATH/qwen2.5-coder-14b-instruct-q4_k_m.gguf"
```

`--jinja` matters. It makes llama-server use the chat template with tool support, which stops most Hermes-tag leaks at the source. proxytool handles what's left: fenced or bare-JSON leaks, malformed JSON, and history shapes that cause `400`s.

Using MLX instead?

```sh
python -m mlx_lm.server --model mlx-community/Qwen2.5-Coder-14B-Instruct-4bit --port 8181
```

**2. Start proxytool:**

```sh
proxytool serve --upstream-url http://127.0.0.1:8181
```

It binds `127.0.0.1:8180` by default.

**3. Check it's alive:**

```sh
proxytool health && proxytool models
```

**4. Try it without an agent:**

```sh
curl -s http://127.0.0.1:8180/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen","stream":false,
       "messages":[{"role":"user","content":"list the files"}],
       "tools":[{"type":"function","function":{"name":"list_dir","parameters":{"type":"object","properties":{"path":{}}}}}]}' \
  | jq '.choices[0].message'
```

If the model leaked, you'll see `tool_calls` filled in and `content` free of stray JSON.

## Pointing your agent at it

Either way, the base URL is proxytool's `/v1`, never the model server directly.

### opencode

Two ready-made configs in [`examples/opencode/`](../../examples/opencode/): `opencode-gguf.jsonc` (llama-server, model id `proxytool/qwen-gguf`) and `opencode-mlx.jsonc` (mlx_lm.server, `proxytool/qwen-mlx`).

```sh
export OPENCODE_CONFIG=./examples/opencode/opencode-gguf.jsonc
opencode
```

### pi

[pi](https://pi.dev/) works the same way.

```sh
curl -fsSL https://pi.dev/install.sh | sh          # needs Node 22.19+
mkdir -p ~/.pi/agent && cp examples/pi/models.json ~/.pi/agent/models.json

pi --provider proxytool --model qwen-mlx           # interactive
pi --provider proxytool --model qwen-mlx -p "..."  # one-shot
pi --list-models proxytool                         # which models can it see?
```

Swap `qwen-mlx` for whichever id is in your `models.json`. Inside a session, `/model` switches models without a restart. The `compat` flags in the example file (`supportsDeveloperRole`, `supportsReasoningEffort`) keep pi's requests friendly with local servers.

## The CLI

| Command | What it does |
| --- | --- |
| `serve` | Run in the foreground |
| `start` | Run as a daemon (same flags as `serve`); returns once `/health` answers |
| `stop` | Stop a `start`ed daemon (SIGTERM, a grace period, then SIGKILL) |
| `status` | Running, stopped or stale, read from the state file |
| `health` | JSON probe of proxytool plus each registered backend |
| `models` | JSON list of each upstream's model ids (what routing matches on) |
| `config` | `config print` shows what would actually be used; `config check` just validates |
| `version` | Prints the version |

`proxytool serve --help` lists every flag. `proxytool --help` never starts anything. State lives in `$LTP_STATE_PATH` (default `/tmp/proxytool-serve-state.json`); the daemon's log sits next to it.

## Settings

Every flag has an env-var twin. Order of precedence: **CLI flag > env var > config file > built-in default**.

| Flag | Env var | Default | What it does |
| --- | --- | --- | --- |
| `--host` | `LTP_HOST` | `127.0.0.1` | Bind host |
| `--port` | `LTP_PORT` | `8180` | Bind port |
| `--upstream-url URL` | `LTP_UPSTREAM_URL` | `http://127.0.0.1:8181` | Single backend to forward to |
| `--upstream NAME=URL` (repeatable) | `LTP_UPSTREAM_<NAME>` | none | Register a routed backend |
| `--upstream-models NAME=m1,m2` (repeatable) | `LTP_UPSTREAM_<NAME>_MODELS` | none | Model ids routed to that backend |
| `--default-backend NAME` | `LTP_DEFAULT_BACKEND` | first registered | Fallback when no model matches |
| `--model-map JSON` / `--model-map-file PATH` | `LMW_MODEL_MAP` / `LMW_MODEL_MAP_FILE` | none | Client alias → real model id |
| `--upstream-api-key TOKEN` | `LMW_UPSTREAM_API_KEY` | none | Static bearer token sent upstream |
| `--upstream-timeout SECS` | `LMW_UPSTREAM_TIMEOUT` | `300` | Upstream read timeout (covers cold model loads) |
| `--loop-nudge N` / `--loop-abort N` | `LMW_LOOP_NUDGE` / `LMW_LOOP_ABORT` | `3` / `6` | LoopGuard thresholds (`0` = off) |
| `--trace` | `LMW_TRACE` / `LTP_TRACE` | off | Dump every payload transit point to stderr |
| `--log-level debug\|info\|warning\|error` | `LTP_LOG_LEVEL` / `LMW_LOG_LEVEL` | `info` | Log level, for proxytool *and* toolpatch |
| `--quiet` | none | off | Same as `--log-level error` |
| `--config PATH` | `LTP_CONFIG` | discovery | TOML config file |

A few extras that only exist as env vars: `LMW_BACKEND` (`gguf` \| `mlx` \| `omlx` \| `fake`, default `gguf`) tells the repair layer which server dialect it's talking to, and `LMW_TRACE_FILE` writes trace output to a file instead of stderr.

The old `LMW_*` names still work everywhere. If both are set, `LTP_*` wins.

### Config file

Rather not type flags every time? Drop in a TOML file and run bare commands:

```sh
mkdir -p ~/.config/proxytool
cp examples/proxytool/proxytool.toml ~/.config/proxytool/config.toml   # or ./proxytool.toml

proxytool start
proxytool health && proxytool models
proxytool status && proxytool stop
```

proxytool looks for a config in this order, first hit wins: `--config PATH`, `$LTP_CONFIG`, `./proxytool.toml`, `./.proxytool.toml`, `$XDG_CONFIG_HOME/proxytool/config.toml`, `~/.config/proxytool/config.toml`.

Sections are `host` / `port` / `log_level` / `default_backend` at the top level, then `[upstream]`, `[routing.<name>]`, `[model_map]`, `[loopguard]` and `[trace]`. The annotated [`examples/proxytool/proxytool.toml`](../../examples/proxytool/proxytool.toml) has the lot.

The file is strictly validated, so typos fail fast with a helpful hint rather than silently doing nothing:

```
proxytool: config error: proxytool.toml:
  - unknown key 'prot' (top level) — did you mean 'port'?
  - unknown key 'timeot' in [upstream] — did you mean 'timeout'?
  - default = true on more than one routing entry: gguf, mlx
```

Not sure why a setting isn't landing? Ask proxytool what it would use, without starting anything:

```sh
proxytool config print    # discovery, validation and every effective setting
proxytool config check    # validation only (handy in CI)
```

A row marked `env (pre-existing)` means your environment already set that key, so the file didn't override it.

## More than one model server

Register several backends and proxytool routes each request by the `model` id the client sends, falling back to a default:

```sh
proxytool serve \
  --upstream gguf=http://127.0.0.1:8181 --upstream-models gguf=qwen-gguf,qwen \
  --upstream mlx=http://127.0.0.1:8281  --upstream-models mlx=qwen-mlx \
  --default-backend mlx
```

Exactly one backend registered? It gets everything. Several? Chat requests are matched on the model list first, then the default. `/health`, `/v1/models` and everything else still go to the primary URL (`--upstream-url`, or the first backend). Run with `--trace` and you'll see each routing decision logged as `MW.ROUTE`.

Same thing via env vars:

```sh
LTP_UPSTREAM_GGUF=http://127.0.0.1:8181 \
LTP_UPSTREAM_GGUF_MODELS=qwen-gguf,qwen \
LTP_UPSTREAM_MLX=http://127.0.0.1:8281 \
LTP_DEFAULT_BACKEND=mlx \
proxytool serve
```

## Using it with llama-swap

[llama-swap](https://github.com/mostlygeek/llama-swap) starts model servers on demand from a YAML file and unloads them when idle. proxytool sits in front of it and keeps doing the repairs.

**1. Declare your models:**

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

**2. Start llama-swap, then proxytool:**

```sh
llama-swap --config ~/.llama-swap/config.yml            # listens on :8080
proxytool serve --upstream-url http://127.0.0.1:8080 \
  --upstream-timeout 600                                # covers cold model loads
```

**3. Point your agent at proxytool's `/v1`**, not at llama-swap. Then `proxytool health && proxytool models` to sanity-check.

Some things worth knowing:

- **Aliases:** client names like `qwen-gguf` are resolved via `LMW_MODEL_MAP` first, then by probing the upstream's `/v1/models`. With several models loaded, the closest name match wins.
- **Cold starts:** the first request after a quiet spell can be slow while llama-swap loads the model. That's what the timeout is for. Keep `--loop-abort` above 3 or 4 so a retry storm during a cold start doesn't look like a loop.
- **Auth:** the client's `Authorization` header is forwarded per request. `--upstream-api-key` adds a static token for gated gateways or `llama-server --api-key`.
- **Streaming:** responses stream straight through. The only buffering is while a leak might be forming, plus reassembling split SSE lines.

## LoopGuard

Agents occasionally get stuck sending the identical request again and again. LoopGuard counts consecutive byte-identical requests (same sanitised messages and tools). After `--loop-nudge` repeats it appends a nudge telling the model to stop. After `--loop-abort` it stops forwarding and returns a clean completion explaining why ("LoopGuard: request aborted after N identical attempts"). Any different request resets the count.

## Tests

```sh
uv run pytest          # unit, regression and proxy integration tests
uv run ruff check .
uv run pyright
uv run nox -s coverage # branch coverage across both modules
```

Found a new way your model mangles a tool call? Add it as a regression fixture pair. The how-to is in `packages/toolpatch/tests/regression/fixtures/README.md`.

### Live tests (opt-in)

These need real things running (a model behind proxytool on `:8180`, and the agent CLIs installed). They skip themselves otherwise, so `pytest` stays green without them. Switch them on with `LMW_LIVE=1`:

```sh
LMW_LIVE=1 uv run pytest apps/proxytool/tests/test_live_qwen.py -q       # talks to the model directly
LMW_LIVE=1 uv run pytest apps/proxytool/tests/test_live_opencode.py -q   # real opencode CLI
LMW_LIVE=1 uv run pytest apps/proxytool/tests/test_live_pi.py -q         # real pi CLI
```

- **Qwen suite:** checks nothing leaks (no tag debris, no raw JSON beside a repaired call), streaming and non-streaming. If your model does something new, it prints a capture block ready to paste into the fixtures. Override the target with `LMW_MIDDLEWARE_URL` (default `http://127.0.0.1:8180`) and `LMW_LIVE_MODEL`.
- **opencode and pi:** each runs the real CLI in a throwaway sandbox with a temporary config (`OPENCODE_CONFIG` / `PI_MODELS_FILE`), so your own settings are never touched. They check a structured tool call made it through the agent's parser with no leakage. Override binaries with `LMW_OPENCODE_BIN`, `LMW_PI_BIN` and `LMW_LIVE_MODEL_PI` (defaults to `qwen-mlx`).
