x<p align="center">
  <img src="public/assets/images/proxytoolpatch.webp" alt="proxytoolpatch" width="220">
</p>

# proxytoolpatch workspace

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)

[![publish toolpatch + proxytool](https://github.com/dave-holden/proxytoolpatch/actions/workflows/publish.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/publish.yml)

[![Python 3.10–3.14](https://img.shields.io/badge/python-3.10%E2%80%933.14-blue)](https://www.python.org)

[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)

[![PyPI proxytool](https://img.shields.io/badge/PyPI-proxytool-blue)](https://pypi.org/project/proxytool/)
[![PyPI toolpatch](https://img.shields.io/badge/PyPI-toolpatch-blue)](https://pypi.org/project/toolpatch/)

## Why I built this

I wanted to run a coding LLM locally without making my MacBook Pro M1 with 32GB
of RAM regret its life choices. After a fair bit of research, Qwen kept coming
up in the recommendations — specifically `Qwen2.5-Coder-14B-Instruct-GGUF` —
so I installed `llama.cpp` and downloaded the model with `hf`.

The first test was deliberately simple: ask the model to list the contents of
the current directory. With both **pi** and **opencode**, the result was not a
working tool call. Instead, the model returned a JSON object containing the
`ls` command. Technically, the model had produced something useful; practically,
the agent clients had no idea what to do with it.

Here is the problem in pictures. Without `proxytool`, both clients show the
leaked tool call as an error or as raw output:

| pi | opencode |
|---|---|
| ![pi showing a tool-call mismatch](https://private-us-east-1.manuscdn.com/sessionFile/pEE3MQ52MEI8g66Xm1hBVU/sandbox/BbqRhcS0yGr84nHB4CDL45-images_1792358456337_na1fn_L2hvbWUvdWJ1bnR1L3Byb3h5dG9vbHBhdGNoL3B1YmxpYy9hc3NldHMvaW1hZ2VzL3BpLXNob3dpbmctdG9vbC1jYWxsLW1pc21hdGNo.webp?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvcEVFM01RNTJNRUk4ZzY2WG0xaEJWVS9zYW5kYm94L0JicVJoY1MweUdyODRuSEI0Q0RMNDUtaW1hZ2VzXzE3OTIzNTg0NTYzMzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzQnliM2g1ZEc5dmJIQmhkR05vTDNCMVlteHBZeTloYzNObGRITXZhVzFoWjJWekwzQnBMWE5vYjNkcGJtY3RkRzl2YkMxallXeHNMVzFwYzIxaGRHTm8ud2VicCIsIkNvbmRpdGlvbiI6eyJEYXRlTGVzc1RoYW4iOnsiQVdTOkVwb2NoVGltZSI6MTc5NDcwMDgwMH19fV19&Key-Pair-Id=K2QY5QTL8JSY6C&Signature=MEQCIFnIBe7mbV1f7HXHpNBt4IXNKV77jjFTLCZNr5Q8SkNAAiAn-1y1l~Q65kEcHPPb5nCaPEpZkifou~NB2i20KXS6mg__) | ![opencode showing a tool-call mismatch](https://private-us-east-1.manuscdn.com/sessionFile/pEE3MQ52MEI8g66Xm1hBVU/sandbox/BbqRhcS0yGr84nHB4CDL45-images_1792358456337_na1fn_L2hvbWUvdWJ1bnR1L3Byb3h5dG9vbHBhdGNoL3B1YmxpYy9hc3NldHMvaW1hZ2VzL29wZW5jb2RlLXNob3dpbmctdG9vbC1jYWxsLW1pc21hdGNo.webp?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvcEVFM01RNTJNRUk4ZzY2WG0xaEJWVS9zYW5kYm94L0JicVJoY1MweUdyODRuSEI0Q0RMNDUtaW1hZ2VzXzE3OTIzNTg0NTYzMzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzQnliM2g1ZEc5dmJIQmhkR05vTDNCMVlteHBZeTloYzNObGRITXZhVzFoWjJWekwyOXdaVzVqYjJSbExYTm9iM2RwYm1jdGRHOXZiQzFqWVd4c0xXMXBjMjFoZEdOby53ZWJwIiwiQ29uZGl0aW9uIjp7IkRhdGVMZXNzVGhhbiI6eyJBV1M6RXBvY2hUaW1lIjoxNzk0NzAwODAwfX19XX0_&Key-Pair-Id=K2QY5QTL8JSY6C&Signature=MEQCIA3NkvoCBFMCfWZQOX7jmNB43GUds7s8Ol-IPSQjU3rDAiA5jHiV5DuTX1eXJwuOUmQBK9fZScFfvzbjhhW6ttQN7g__) |

Put `proxytool` in front of the model and the same prompt produces a proper
directory listing:

| pi | opencode |
|---|---|
| ![pi showing directory contents](https://private-us-east-1.manuscdn.com/sessionFile/pEE3MQ52MEI8g66Xm1hBVU/sandbox/BbqRhcS0yGr84nHB4CDL45-images_1792358456337_na1fn_L2hvbWUvdWJ1bnR1L3Byb3h5dG9vbHBhdGNoL3B1YmxpYy9hc3NldHMvaW1hZ2VzL3BpLXNob3dpbmctZGlyZWN0b3J5LWNvbnRlbnQ.webp?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvcEVFM01RNTJNRUk4ZzY2WG0xaEJWVS9zYW5kYm94L0JicVJoY1MweUdyODRuSEI0Q0RMNDUtaW1hZ2VzXzE3OTIzNTg0NTYzMzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzQnliM2g1ZEc5dmJIQmhkR05vTDNCMVlteHBZeTloYzNObGRITXZhVzFoWjJWekwzQnBMWE5vYjNkcGJtY3RaR2x5WldOMGIzSjVMV052Ym5SbGJuUS53ZWJwIiwiQ29uZGl0aW9uIjp7IkRhdGVMZXNzVGhhbiI6eyJBV1M6RXBvY2hUaW1lIjoxNzk0NzAwODAwfX19XX0_&Key-Pair-Id=K2QY5QTL8JSY6C&Signature=MEYCIQC65mE5svHEd3b1MCHlm6ek9PuG1dSdC5xwO1hWEJFAmAIhAPbZjO3p9lAkLGd1vAGaII7PcvQfE-rAQ81SBjeJM6L~) | ![opencode showing directory contents](https://private-us-east-1.manuscdn.com/sessionFile/pEE3MQ52MEI8g66Xm1hBVU/sandbox/BbqRhcS0yGr84nHB4CDL45-images_1792358456337_na1fn_L2hvbWUvdWJ1bnR1L3Byb3h5dG9vbHBhdGNoL3B1YmxpYy9hc3NldHMvaW1hZ2VzL29wZW5jb2RlLXNob3dpbmctZGlyZWN0b3J5LWNvbnRlbnQ.webp?Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9wcml2YXRlLXVzLWVhc3QtMS5tYW51c2Nkbi5jb20vc2Vzc2lvbkZpbGUvcEVFM01RNTJNRUk4ZzY2WG0xaEJWVS9zYW5kYm94L0JicVJoY1MweUdyODRuSEI0Q0RMNDUtaW1hZ2VzXzE3OTIzNTg0NTYzMzdfbmExZm5fTDJodmJXVXZkV0oxYm5SMUwzQnliM2g1ZEc5dmJIQmhkR05vTDNCMVlteHBZeTloYzNObGRITXZhVzFoWjJWekwyOXdaVzVqYjJSbExYTm9iM2RwYm1jdFpHbHlaV04wYjNKNUxXTnZiblJsYm5RLndlYnAiLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE3OTQ3MDA4MDB9fX1dfQ__&Key-Pair-Id=K2QY5QTL8JSY6C&Signature=MEYCIQD0u-P9OWtDrANcRWZJk0~~IAXpIxloykIpZI~EzJFg7gIhAPfimyUqHv83A0P6p3IoBDnfKvo3jud4ktUX2ovh~GT7) |

That is the whole idea behind this repository: put a small compatibility layer
between the agent and the local model, clean up the leaked tool call, and let
the agent carry on. The implementation is split into two parts:

- **`proxytool`** is the runnable proxy and CLI.
- **`toolpatch`** is the Python library that does the repair work.

Technically, this is a workspace for **agent-ready local LLM serving**: run a
small model with a standard OpenAI-compatible server, put this in front of it,
and agent clients (opencode, pi, curl, or anything speaking the OpenAI dialect)
get clean structured tool calls — even when the model leaks them as raw text.

```
┌──────────────────────────────────────────────────────────────────┐
│  Agent clients:  opencode · pi · curl · any OpenAI-compatible    │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP (OpenAI dialect)
┌──────────────────────────▼───────────────────────────────────────┐
│  proxytool  (apps/proxytool) — the runnable app                  │
│  FastAPI + `proxytool` CLI: bind, supervise, route, sanitize     │
│  requests, repair leaked tool calls (streaming + non-streaming), │
│  guard against repetition loops — all repair logic via the       │
│  library below                                                   │
└──────────────────────────┬───────────────────────────────────────┘
                           │ in-process import
┌──────────────────────────▼───────────────────────────────────────┐
│  toolpatch  (packages/toolpatch) — the repair library            │
│  Pure, zero-I/O Python: leak detection (Hermes / fenced / bare   │
│  JSON), JSON healing, SSE stream healing, history normalization  │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP (OpenAI dialect)
┌──────────────────────────▼───────────────────────────────────────┐
│  Backend servers: llama-server (GGUF) · mlx_lm.server · omlx ·   │
│  llama-swap · any OpenAI-compatible local inference server       │
└──────────────────────────────────────────────────────────────────┘
```

## Modules

| Module | Kind | What it provides | Docs |
|---|---|---|---|
| `apps/proxytool` | app | The proxy server + `proxytool` CLI: serve, supervise backends, route by model id, repair leaked tool calls in-flight | [`apps/proxytool/README.md`](apps/proxytool/README.md) |
| `packages/toolpatch` | library | Pure-logic repair engine: `extract_tool_calls`, `heal_json`, `StreamHealer`, `normalize_chat_request` — embeddable by any Python server | [`packages/toolpatch/README.md`](packages/toolpatch/README.md) |
| `examples/` | runnable examples | Copy-paste config files: proxytool TOML, opencode + pi client wiring, llama-swap upstream | [`examples/README.md`](examples/README.md) |
| `scripts/` | local tooling | Personal dev scripts (dev rig, full test matrix, fake upstream) — intentionally untracked, not needed to run the modules; the git hooks that used to live here are tracked in [`.githooks/`](.githooks/) | — |

The naming: `toolpatch` is the published library name; both components work for any small local model, not just
Qwen.

## Install

Workspace (development):

```bash
uv sync --all-packages     # one shared root .venv, editable across modules
```

Once published, `pip install proxytool` will be enough for the app — it
declares the library as a dependency, so installing the app pulls the
library too. The library alone is `pip install toolpatch` (published as `toolpatch`; previously
a rename candidate).

## Usage

The `proxytool` command is a console script registered in the app package
(`[project.scripts]` in `apps/proxytool/pyproject.toml`), so any install of
the app — workspace `uv sync`, `pip install proxytool`, or
`uv tool install proxytool` — puts it on your PATH. Full CLI reference:
[`apps/proxytool/README.md`](apps/proxytool/README.md).

**What it is, one line:** `proxytool` is a FastAPI proxytool + CLI that sits
in front of standard OpenAI-compatible local model servers, rewrites handler
aliases, repairs garbled tool calls in-flight via the `toolpatch` library,
and aborts asking-loops via LoopGuard.

**Scope, stated clearly:** proxytool only looks after itself. It fronts
model servers that are *already running*; owning their process lifecycle is
deliberately out of scope — that's the job of llama-swap (multi-model start /
stop from YAML), or your compose/systemd unit.

### Startup order

For the common setups the pieces come up in *this* order:

1. **LLM server(s)** (llama.cpp, mlx_lm, oMLX) — must be listening first
2. **llama-swap** — if you want on-demand multi-model swaps, start it after
   its backends
3. **proxytool** — last server piece: it health-checks its upstream at
   startup and holds traffic until the backend answers
4. **Agent client** (opencode, pi, …) — only needs proxytool's endpoint URL

### 1 · Start the LLM server yourself

```bash
# llama.cpp with the qwen tool-call template
llama-server --host 127.0.0.1 --port 8181 --jinja \
  --model "$MPATH/qwen2.5-coder-14b-instruct-q4_k_m.gguf" &

# or mlx_lm.server
python -m mlx_lm.server --model mlx-community/Qwen2.5-Coder-14B-Instruct-4bit --port 8181

# (no backend env needed: proxytool auto-detects the dialect per request
#  from the model id — an mlx model id gets tool_choice stripped upstream)
```

### 2 · Start llama-swap (scenario B: llama-swap owns the backends)

llama-swap decides which model to spin up from its own YAML — proxytool just
stands in front of it and keeps the repair/aliasing per request:

```bash
llama-swap --config llama-swap/config.yml           # listens on :8080
proxytool serve --upstream-url http://127.0.0.1:8080 \
  --model-map '{"qwen-gguf":"<real llama-swap model id>"}'
```

Equivalent env: `PTP_UPSTREAM_URL=http://127.0.0.1:8080`,
`PTP_MODEL_MAP='{…}'`. See `examples/llama-swap/config.yml` for a working
upstream file.

Then start proxytool itself — foreground, or daemonized with its own
lifecycle commands (this self-management is what proxytool *does* own):

```bash
proxytool serve --upstream-url http://127.0.0.1:8080      # foreground

# or the daemon form: returns once its /health answers
proxytool start --upstream-url http://127.0.0.1:8080
proxytool status && proxytool health && proxytool models
uv run proxytool status && uv run proxytool health && uv run proxytool models
proxytool stop
```

### 3 · Start the agent client

```bash
# opencode — point it at proxytool via one env var
export OPENCODE_CONFIG=examples/opencode-gguf.jsonc   # or opencode-mlx.jsonc
opencode

# pi — copy the provider file once, then run
mkdir -p ~/.pi/agent && cp examples/pi/models.json ~/.pi/agent/models.json
pi --provider proxytool --model qwen-mlx -p "list the files in ."

# smoke-test without any agent
PTP_TRACE=1 uv run uvicorn proxytool.main:app --port 8180   # dev server
```

The configs point their baseURL at `http://127.0.0.1:8180/v1` (proxytool).
Aliases are rewritten server-side; clients stay oblivious.

### 4 · One middle layer, several backends (scenario C)

When not using llama-swap but more than one model server, let proxytool
route by model id and pick the fallback:

```bash
proxytool serve \
  --upstream gguf=http://127.0.0.1:8181 --upstream-models gguf=qwen-gguf \
  --upstream mlx=http://127.0.0.1:8281 --upstream-models mlx=qwen-mlx \
  --default-backend mlx
```

Full env-var table, precedence rules, config schema and diagnostics:
[`apps/proxytool/README.md`](apps/proxytool/README.md). What gets repaired
when a model leaks: [`packages/toolpatch/README.md`](packages/toolpatch/README.md).

## Tests & checks

```bash
uv run pytest          # unit + regression + proxy integration
uv run ruff check .
uv run pyright
uv run nox             # lint + full multi-Python matrix (3.10–3.14, both modules)
```

Nox (`noxfile.py` at the root) drives the same per-version test sessions CI
uses — every slice installs the workspace packages as wheels under the target
interpreter in a throwaway uv env — and CI runs those nox sessions directly,
so `noxfile.py` is the single source of truth for what “checked” means.
`uv run nox` locally is exactly what `test-matrix` in CI tests.

Live model / real-client tests are opt-in (`PTP_LIVE=1`) and auto-skip —
see [`apps/proxytool/README.md`](apps/proxytool/README.md#tests--checks).

Matrix / hooks / CI / PR setup:

- Multi-version matrix: `uv run nox -s test` (Python 3.10–3.14, both modules) —
  the same nox sessions CI runs (a local test-matrix helper may exist in your
  untracked tooling; the repo itself needs only nox)
- Git hooks: tracked in `.githooks/` — activate once per clone with
  `./.githooks/install.sh`, then pre-commit runs lint + default-python tests,
  pre-push runs the same; with no `core.hooksPath` git skips them
- CI mirrors all of it on every PR (`.github/workflows/test.yml`); publishing
  is tag-driven and test-gated (`.github/workflows/publish.yml`)
- PR/branch-protection steps + daemon CLI (start/stop/status/health/models):
  [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md)

## Design history

Design rationale and release notes are maintained as local-only files,
deliberately outside the repo, so this README stays authoritative for a
fresh clone. What the repo itself ships is the git tag history — pushing
a version tag publishes both wheels through
`.github/workflows/publish.yml` (test-gated).
