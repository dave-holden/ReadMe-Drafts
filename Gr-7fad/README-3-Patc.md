<p align="center">
  <img src="../../public/assets/images/toolpatch.webp" alt="toolpatch" width="200">
</p>

# toolpatch

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)
[![PyPI toolpatch](https://img.shields.io/badge/PyPI-toolpatch-blue)](https://pypi.org/project/toolpatch/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org)
[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)

This is the pure repair library that proxytool uses under the hood.

Local models love to emit tool calls as text inside `content` instead of the proper `tool_calls` field. Feed that garbled reply to toolpatch and you get clean structured tool calls back.

It’s deliberately zero-I/O — no sockets, no event loop, no CLI. Just pure Python you can embed anywhere (or let proxytool do it for you).

```
proxytool  (the runnable app: HTTP, routing, supervision)
    └─ imports
toolpatch (this library: pure text repair, no I/O)
```

## Install

```bash
pip install toolpatch          # once on PyPI
# or
uv add toolpatch
```

In this workspace:

```bash
uv sync --all-packages
```

## Quick use

```python
from toolpatch import extract_tool_calls

reply = 'Sure. <tool_call>{"name": "list_dir", "arguments": {"path": "."}}</tool_call>'

ex = extract_tool_calls(reply, tool_names={"list_dir"})
if ex:
    ex.content      # "Sure."  (leak gone)
    ex.tool_calls   # [{'id': 'call_…', 'type': 'function',
                    #   'function': {'name': 'list_dir', 'arguments': '{"path": "."}'}}]
```

That’s the whole loop for a completed reply. For streaming use `StreamHealer`; for outbound request cleanup use `normalize_chat_request`.

## What it repairs

Three leaked shapes (all merged, in document order):

1. **Hermes-style XML function tags** — the form Qwen and several other open models emit when they leak  
2. **Fenced JSON** — ```json … ``` blocks  
3. **Bare JSON object** — raw `{"name": …, "arguments": …}` sitting in prose  

Plus the supporting work:

- JSON healing (trailing commas, smart quotes, raw newlines, fences…)
- History normalisation (the shapes llama-server rejects with 400s)
- Streaming repair with bounded buffering so time-to-first-token stays good

## API

Small public surface, all re-exported from `toolpatch`:

```python
from toolpatch import (
    extract_tool_calls,
    heal_json,
    StreamHealer,
    normalize_chat_request,
    ExtractedToolCalls,   # dataclass: content, tool_calls, source
)
```

### `extract_tool_calls(content, tool_names=None, allow_unterminated_hermes=False)`

Returns `ExtractedToolCalls | None`. Runs all three detectors, merges distinct calls in document order, returns `None` for clean prose.

### `heal_json(text) -> str | None`

Repairs the common model JSON messes. Returns the fixed string or `None` if it’s beyond saving.

### `StreamHealer(max_buffer=512, tool_names=None)`

Streaming version. Feed it content deltas; it yields safe `content` events and a single `tool_calls` event at the end (or when the buffer forces promotion). Bounded buffering keeps TTFT intact and never forwards raw leak text.

```python
healer = StreamHealer()

for delta in sse_content_deltas:
    for event in healer.feed(delta):
        if event["type"] == "content":
            # safe to forward
            ...
        elif event["type"] == "tool_calls":
            # one frame containing every call seen this turn
            ...

for event in healer.finish():
    ...
```

### `normalize_chat_request(payload, backend="gguf") -> (dict, list[str])`

Deep-copies and sanitises an outbound chat-completion request. `backend="mlx"` also strips `tool_choice`. Returns the cleaned payload plus a list of human-readable notes about what was fixed.

## Embedding

Anything that can import Python can use this:

- Open WebUI Pipelines
- LiteLLM custom callbacks
- Your own FastAPI / FastMCP / custom proxy
- Plain OpenAI-SDK scripts

Non-Python tools (Ollama, llama-server internals, etc.) can’t import a Python library — put **proxytool** in front of them instead.

## Relationship to the app

One-way dependency only:

- `apps/proxytool` imports this library
- This library never imports the app, never opens sockets, and its test suite is fully offline

Real-world misformattings captured from live models live as regression fixtures under `tests/regression/fixtures/`.

## Logging

Silent by default. It uses the stdlib `logging` system but never configures handlers. To turn diagnostics on:

```python
from toolpatch import configure_logger
configure_logger("toolpatch", "debug")
```

(proxytool’s `--log-level debug` already does this for you.)

## Status

`v2.0.0`. Currently depends only on `pydantic` (candidate for stdlib-only later).
