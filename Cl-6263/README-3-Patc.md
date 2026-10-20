![toolpatch](public/assets/images/toolpatch.png)

# toolpatch

[![test & checks](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml/badge.svg)](https://github.com/dave-holden/proxytoolpatch/actions/workflows/test.yml)
[![PyPI toolpatch](https://img.shields.io/badge/PyPI-toolpatch-blue)](https://pypi.org/project/toolpatch/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org)
[![code style: ruff](https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json)](https://github.com/astral-sh/ruff)

**The fix-it library: hand it a mumbled tool call from a local model, get a clean structured one back.**

Local models sometimes write their tool call as text inside the chat message instead of the structured format your software can run. `toolpatch` takes that reply and rebuilds the tool call properly.

It's pure Python with zero I/O: no sockets, no event loop, no CLI. Nothing runs on its own. You either import it into your own code or let [**proxytool**](../../apps/proxytool/README.md) use it for you.

```
proxytool   (the app: HTTP serving, routing, a CLI)
   └── imports
toolpatch   (this library: text repair, nothing else)
```

Just want it working with your agent? You probably want `proxytool`, not this. Still here? Good, read on.

## Install

```sh
pip install toolpatch       # or: uv add toolpatch
```

Working from a clone of the monorepo? `uv sync --all-packages` gives you an editable install of both modules.

## Quick use

One finished (non-streaming) model reply in, clean result out:

```python
from toolpatch import extract_tool_calls

reply = 'Sure. <tool_call>{"name": "list_dir", "arguments": {"path": "."}}</tool_call>'

ex = extract_tool_calls(reply, tool_names={"list_dir"})
if ex:
    ex.content      # "Sure."  (clean prose, leak gone)
    ex.tool_calls   # [{'id': 'call_…', 'type': 'function',
                    #   'function': {'name': 'list_dir', 'arguments': '{"path": "."}'}}]
```

That's the whole loop for a finished reply. For streaming replies use `StreamHealer`, and for cleaning up outgoing requests use `normalize_chat_request`. Both are in [the API section](#the-api).

## What it fixes

When a model's chat template is missing or mismatched, some local servers put the tool call *inside* `choices[*].message.content` instead of the `tool_calls` field. toolpatch spots three shapes of leak and merges them all into one result, in document order:

1. **Hermes-style tags**: a `<tool_call>` block wrapping a JSON payload. It's what Qwen and several other open models produce when they leak.
2. **Fenced JSON**: a `` ```json `` code block.
3. **Bare JSON**: a raw `{"name": …, "arguments": …}` sitting in the prose. It checks the keys and heals the payload before accepting it.

And the supporting repairs:

- **JSON healing:** trailing commas, raw newlines inside strings, smart quotes, code fences.
- **History normalisation:** merges assistant-message shapes that llama-server-style chat templates reject with a `400` (leaked-JSON "twins" of repaired tool calls, adjacent assistant turns).
- **Streaming:** `StreamHealer` does the same job on SSE deltas, without delaying the first token.

### How streaming works

`StreamHealer` isn't "buffer everything until we've got a full match". It only holds text back while a leak *might* be forming:

- Clean prose goes out as soon as nothing looks like the start of a leak.
- The last 10 characters are always held back, so an opener split across two deltas (like `<tool_` then `call>`) is judged whole once the next delta lands. (10 = the 11-character `<tool_call>` opener minus one.)
- Leaks that show up after a reasoning block are caught too, not just ones at the end of the buffer. That's the Qwen2.5-on-MLX pattern that made me add it.
- Once a leak is confirmed it switches to collecting mode and waits until the end of the stream to promote it. That means several calls in a row, even mixed families, arrive in **one** `tool_calls` frame rather than the first call being promoted and the rest dropped.
- Whitespace between two leaked calls survives as prose. Only padding outside the leak is stripped.
- A cap of 4× `max_buffer` stops a never-ending ramble from holding things up forever.

## The API

It's a deliberately small surface: three callables, the stream helper and one dataclass, all importable from `toolpatch`.

```python
from toolpatch import extract_tool_calls, heal_json, StreamHealer, normalize_chat_request
from toolpatch import ExtractedToolCalls  # dataclass: content, tool_calls, source
```

### `extract_tool_calls(content, tool_names=None, allow_unterminated_hermes=False)`

Returns `ExtractedToolCalls | None`.

Runs all three detectors over the text and merges every distinct call into one result, in document order. Overlapping or nested matches collapse to the outer one. Clean prose returns `None` (nothing to do). On a hit you get the cleaned content plus the structured tool calls.

- `tool_names={...}` only accepts calls whose `name` is in the set. Anything else stays as prose.
- `allow_unterminated_hermes=True` is for streaming: it tolerates one half-finished Hermes block at the end of the text.
- `ExtractedToolCalls.source` tells you which family matched (`hermes`, `fenced` or `raw_json`), or `mixed` if there was more than one.

```python
ex = extract_tool_calls('Sure. ' + hermes_block)
ex.content                                  # 'Sure.' (leak text removed)
ex.source                                   # 'hermes'
ex.tool_calls[0]['function']['name']        # 'list_dir'
```

### `heal_json(text) -> str | None`

Repairs malformed JSON from a model: strips code fences, straightens smart quotes, drops trailing commas, escapes raw newlines that broke a string. Returns the repaired JSON string, or `None` if it's beyond saving.

### `StreamHealer(max_buffer=512, tool_names=None)`

For streaming. Feed it content deltas as they arrive; it yields plain-dict events. It buffers only while a leak might be forming (up to `max_buffer`), and the raw leak text is never forwarded, so clients can't end up rendering it twice.

```python
healer = StreamHealer()

for delta in sse_content_deltas:
    for event in healer.feed(delta):
        match event["type"]:
            case "content":
                pass  # safe to forward to the client as-is
            case "tool_calls":
                ...   # event["stop"] is True: one frame per turn,
                      # containing ALL the calls seen

for event in healer.finish():   # flush at the end of the stream
    ...
```

The two event shapes are `{"type": "content", "text": str}` and `{"type": "tool_calls", "tool_calls": [...], "stop": True}`. If there's residual prose between calls, it comes out in the same batch, just before the `tool_calls` event.

### `normalize_chat_request(payload, backend="gguf") -> (dict, list[str])`

Tidies an outgoing chat-completion request without touching the original (it deep-copies). `backend="mlx"` also strips `tool_choice`, which `mlx_lm.server` doesn't support. You get back the cleaned payload plus a list of notes describing each fix. History fixes include merging leaked tool-call text into the following `tool_calls` turn, and merging or deduping adjacent assistant turns. Those are the shapes llama-server templates reject with a `400`.

```python
clean, notes = normalize_chat_request(payload, backend="gguf")
if notes:
    log.info("sanitized request", extra={"notes": notes})
```

## Using it in your own code

Python things that could embed it today:

- **Open WebUI Pipelines:** run `extract_tool_calls` on the model's response before it reaches the client.
- **LiteLLM custom callbacks:** hook extraction into the response path.
- **FastAPI / FastMCP / your own server** in front of a local model: call `normalize_chat_request` before forwarding upstream.
- **Plain OpenAI-SDK scripts:** wrap `chat.completions.create` with `normalize_chat_request` on the way out and `extract_tool_calls` on the way back.

Can't import Python (Ollama, llama-server's C++ internals, llama-swap and friends)? Put **proxytool** in front of them over HTTP. It applies exactly the same repairs.

## Relationship to the app

The dependency only goes one way. `apps/proxytool` imports this library in-process. toolpatch never imports the app, never opens a socket, and its tests run fully offline (`packages/toolpatch/tests`), with no proxy or model needed.

Real-world misformattings captured from live models are kept as regression fixture pairs. See `tests/regression/fixtures/README.md` if you've got a new one to add.

## Logging

The library is silent by default. It logs through the standard `logging` module (logger names like `toolpatch.extractor` and `toolpatch.stream`) but never sets up handlers or levels, because libraries shouldn't. In a bare script you'll see nothing. To switch the diagnostics on:

```python
from toolpatch import configure_logger

configure_logger("toolpatch", "debug")   # or "info" / "warning" / "error"
```

If you're running via proxytool, `--log-level debug` turns this on too. One knob for everything.

## Status

`v2.0.0`. Its only dependency is `pydantic`.
