<!-- Project header -->
<div align="left">
  <h1 style="display: inline; position: top;">proxytoolpatch</h1>
  <img src="public/assets/images/proxytoolpatch.webp" alt="proxytoolpatch logo" width="72" align="left" />
  <p>Fixing tool-call mismatches in local Qwen LLMs.</p>
</div>

Run a coding agent against a small local LLM — without the agent printing raw
JSON at you.

## Why this exists

I can't be the only one, but I had no end of trouble getting a Qwen LLM
working with my machine — a MacBook Pro M1 with 32GB RAM. So here's how it
went:

- I wanted to run an LLM locally for coding.
- My machine is a MacBook Pro M1 with 32GB RAM, so "runs without freezing
  everything else" was a hard requirement.
- After research and asking around, the answer kept coming back as Qwen —
  specifically Qwen2.5.
- llama.cpp for serving — small footprint.
- Downloaded `Qwen2.5-Coder-14B-Instruct-GGUF` with the `hf` Hugging Face
  CLI.
- First prompt: *"list the contents of the current directory."*
- What I got back was a `<tool_call>` mismatch blob — with both **pi** and
  **opencode**.

The model wasn't being stupid — it *was* making the tool call. It just made
it in the wrong place: as `<tool_call>…</tool_call>` text inside the message
body instead of the structured `tool_calls` field the client expects. So the
agent faithfully printed the JSON and stopped.

**Without proxytool — ask for a directory listing, get a JSON object:**

| pi | opencode |
|---|---|
| ![pi showing the tool-call mismatch](../public/assets/images/pi-showing-tool-call-mismatch.webp) | ![opencode showing the tool-call mismatch](../public/assets/images/opencode-showing-tool-call-mismatch.webp) |

**With proxytool in the middle — same prompt, actually works:**

| pi | opencode |
|---|---|
| ![pi showing directory contents](../public/assets/images/pi-showing-directory-content.webp) | ![opencode showing directory contents](../public/assets/images/opencode-showing-directory-content.webp) |

## What's in the repo

Two packages, one job each:

| Package | What it is |
|---|---|
| [`apps/proxytool`](apps/proxytool/README.md) | The thing you run. `proxytool serve` sits between your agent client and your local model server and repairs leaked tool calls before the client ever sees them. |
| [`packages/toolpatch`](packages/toolpatch/README.md) | The fix-it library that does the actual repair — pure Python, zero I/O. proxytool uses it under the hood; you can embed it in your own code too. |

Most people want **proxytool** — head to
[`apps/proxytool/README.md`](apps/proxytool/README.md).

## Development

The git hooks (pre-commit / pre-push quality gates: ruff + pytest) are tracked
in `.githooks/` — activate them once per clone:

```bash
./.githooks/install.sh
```

See [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) for the full local setup
(test matrix, coverage, branch protection).
