# toolpatch package

`packages/toolpatch/` contains reusable logic for recognising and repairing certain malformed tool-call outputs from local language models.

The aim is simple: when a model describes a tool call in text, but the coding agent expects a structured tool call, convert supported output into a form the client can handle.

## The problem it addresses

A model might return something that resembles a tool request as ordinary assistant content—for example, a JSON-like object or a model-specific tool-call marker. That is not necessarily equivalent to an OpenAI-style `tool_calls` entry. If the client receives only text, it generally won't execute the requested tool.

`toolpatch` is intended to handle known output patterns, not arbitrary code or every possible model-specific format.

## What belongs here?

Keep repair and normalisation logic in this package rather than coupling it to FastAPI routes. Depending on the current implementation, this can include:

- Recognising supported textual tool-call formats.
- Extracting a tool name and its arguments.
- Normalising the result into the structured shape expected by the proxy/client.
- Handling malformed or incomplete candidates safely.
- Focused tests for valid, invalid and ambiguous examples.

The exact formats and public API are defined by the current source code and tests. Check those before relying on a particular input syntax or import path.

## Use through the proxy

Most users should start with the runnable app in [`apps/proxytool/`](../../apps/proxytool/README.md). The app integrates this package into the HTTP request/response flow, so you generally don't need to call the repair logic yourself.

## Using it directly

If you want to reuse `toolpatch` in another application, first check the package's `pyproject.toml`, exported symbols and tests for its current installation name and public API. Those are the authoritative references for:

- The import path.
- The callable entry points and accepted input types.
- How a repaired result is represented.
- Which output patterns are supported.
- How parse failures or ambiguous output are reported.

An import example is intentionally omitted until it can match the package's actual exported interface.

## Testing and adding support

When adding support for another model's output format, include tests for at least:

1. A representative valid tool call.
2. Arguments containing strings, escaped characters or nested JSON.
3. Malformed or incomplete output.
4. Text that looks like JSON but should remain ordinary assistant content.
5. The exact structured result expected by the client.

Prefer narrow, explicit matching over permissive parsing. A false positive can be worse than leaving a response untouched: ordinary assistant text should not accidentally become an executable tool call.

## Scope and limitations

- This package repairs supported formats; it does not improve the model's reasoning.
- It cannot reliably infer the intended tool call from arbitrary prose.
- A syntactically valid repair does not mean the tool is safe or appropriate to execute. The client remains responsible for tool execution and permissions.
- Compatibility depends on the model's output, the inference backend and the consuming client.

## Related documentation

- [Repository overview and screenshots](../../README.md)
- [FastAPI proxy application](../../apps/proxytool/README.md)
