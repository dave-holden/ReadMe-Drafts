<!-- Project header -->
<div align="left">
  <h1 style="display: inline; position: top;">proxytoolpatch</h1>
  <img src="public/assets/images/proxytoolpatch.webp" alt="proxytoolpatch logo" width="72" align="left" />
  <p>Fixing tool-call mismatches in local Qwen LLMs.</p>
</div>

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)
[![publish toolpatch + proxytool](https://github.com/dave-holden/proxytoolpatch/actions/workflows/publish.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/publish.yml)
[![Python 3.10–3.14](https://img.shields.io/badge/python-3.10%E2%80%933.14-blue)](https://www.python.org)
[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)
[![PyPI proxytool](https://img.shields.io/badge/PyPI-proxytool-blue)](https://pypi.org/project/proxytool/)
[![PyPI toolpatch](https://img.shields.io/badge/PyPI-toolpatch-blue)](https://pypi.org/project/toolpatch/)

I wanted a coding LLM that would actually run on my MacBook Pro M1 (32 GB) without freezing the machine. After the usual research rabbit-hole, everyone pointed at Qwen — specifically `Qwen2.5-Coder-14B-Instruct-GGUF` via llama.cpp.

It was great… until I pointed an agent at it.

Every time I asked opencode or pi something simple like “list the contents of the current directory”, the model would spit out a JSON tool-call object as plain text instead of a proper structured tool call. The agents just printed the JSON and shrugged.

**Without the proxy** (the mismatch):

| opencode | pi |
|:---:|:---:|
| ![opencode mismatch](../public/assets/images/opencode-showing-tool-call-mismatch.webp) | ![pi mismatch](../public/assets/images/pi-showing-tool-call-mismatch.webp) |

**With the proxy in front** (it just works):

| opencode | pi |
|:---:|:---:|
| ![opencode works](../public/assets/images/opencode-showing-directory-content.webp) | ![pi works](../public/assets/images/pi-showing-directory-content.webp) |

That’s the whole point of this repo.

**In plain English:** local models sometimes *mumble* their tool commands. Your agent asks for a directory listing and gets raw JSON dumped into the chat instead of a clean, executable tool call. This workspace sits in front of any OpenAI-compatible local server and fixes the mumble in-flight. Point opencode, pi, curl, or anything else at one local address and tool calls just work.

```
┌──────────────────────────────────────────────────────────────────┐
│  Agent clients:  opencode · pi · curl · any OpenAI-compatible    │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP (OpenAI dialect)
┌──────────────────────────▼───────────────────────────────────────┐
│  proxytool  (apps/proxytool) — the runnable app                  │
│  FastAPI + CLI: bind, route, sanitize, repair leaked tool calls  │
│  (streaming + non-streaming), stop repetition loops              │
└──────────────────────────┬───────────────────────────────────────┘
                           │ in-process import
┌──────────────────────────▼───────────────────────────────────────┐
│  toolpatch  (packages/toolpatch) — the pure repair library       │
│  Zero I/O: leak detection, JSON healing, stream healing,         │
│  history normalisation                                           │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP
┌──────────────────────────▼───────────────────────────────────────┐
│  Your backend: llama-server · mlx_lm.server · omlx · llama-swap  │
│  or any other OpenAI-compatible local server                     │
└──────────────────────────────────────────────────────────────────┘
```

## Modules

| Module | What it is | Docs |
|---|---|---|
| `apps/proxytool` | The thing you actually run. FastAPI proxy + CLI | [apps/proxytool/README.md](apps/proxytool/README.md) |
| `packages/toolpatch` | The pure-Python repair engine (no sockets, no event loop) | [packages/toolpatch/README.md](packages/toolpatch/README.md) |
| `examples/` | Copy-paste configs for opencode, pi, llama-swap, etc. | [examples/README.md](examples/README.md) |

Works with any small local model, not just Qwen.

## Install

**Development workspace:**

```bash
uv sync --all-packages     # one shared root .venv, editable across modules
```

Once published, `pip install proxytool` is enough for the app — it pulls in the library automatically. The library alone is `pip install toolpatch`.

## Quick start (the common path)

```bash
# 1. Start your model server (example with llama.cpp)
llama-server --host 127.0.0.1 --port 8181 --jinja \
  --model "$MPATH/qwen2.5-coder-14b-instruct-q4_k_m.gguf" &

# 2. Start the proxy in front of it
proxytool serve --upstream-url http://127.0.0.1:8181

# 3. Point your agent at the proxy
export OPENCODE_CONFIG=examples/opencode-gguf.jsonc
opencode
# or
pi --provider proxytool --model qwen-gguf -p "list the files in ."
```

Full install, multi-backend routing, llama-swap, daemon mode, config files, env vars, etc. live in the module READMEs.

### Startup order

1. **LLM server(s)** (llama.cpp, mlx_lm, oMLX) — must be listening first  
2. **llama-swap** (optional) — if you want on-demand multi-model swaps  
3. **proxytool** — last server piece; it health-checks its upstream at startup  
4. **Agent client** (opencode, pi, …) — only needs proxytool’s endpoint URL

### One middle layer, several backends

```bash
proxytool serve \
  --upstream gguf=http://127.0.0.1:8181 --upstream-models gguf=qwen-gguf \
  --upstream mlx=http://127.0.0.1:8281 --upstream-models mlx=qwen-mlx \
  --default-backend mlx
```

## Tests & checks

```bash
uv run pytest          # unit + regression + proxy integration
uv run ruff check .
uv run pyright
uv run nox             # lint + full multi-Python matrix (3.10–3.14)
```

Live model / real-client tests are opt-in (`PTP_LIVE=1`) and auto-skip — see the proxytool README.

## Design history

Design rationale and release notes live outside the repo so this README stays the single source of truth for a fresh clone. Tag history + the publish workflow are what the repo itself ships.
