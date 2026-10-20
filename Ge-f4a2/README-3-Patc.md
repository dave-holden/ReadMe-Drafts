# toolpatch

`toolpatch` is a Python library that handles the nuts and bolts of parsing, detecting, and repairing malformed LLM tool-calling streams. It's the core engine powering `proxytool`.

## What it does

When local LLMs get confused by complex system prompts or token boundaries, their tool calls occasionally bleed out into plain text JSON or broken XML-like structures. `toolpatch` provides the regex patterns, stream buffers, and patch routines needed to intercept these malformed chunks and coerce them back into valid, executable tool calls before the client chokes on them.

## Usage

Drop it into your custom proxy, API middleware, or LLM wrapper pipelines where local model quirks need a bit of straightening out:

```python
from toolpatch import patch_stream

# Example integration snippet
cleaned_stream = patch_stream(raw_llm_output)
```

## Development

Managed within the monorepo workspace using `uv`. Run tests via:

```bash
uv run nox
```