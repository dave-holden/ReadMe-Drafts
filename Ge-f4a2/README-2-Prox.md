# proxytool

`proxytool` is a lightweight proxy app built to sit quietly between your local AI coding agents (`pi`, `opencode`, etc.) and your local LLM backend (like `llama.cpp` or Ollama). 

## Why does this exist?

If you've run local models on modest hardware, you've probably seen this happen: you ask your agent to list a directory, and instead of running `ls`, it spits out a raw JSON object containing the tool call text. The agent's UI breaks, the command fails, and you're stuck nudging the model back on track.

`proxytool` intercepts the response stream, catches these format mismatches in real-time, and patches them using `toolpatch` before they hit your client UI.

## Features

- **In-flight Stream Healing:** Fixes broken tool-call outputs on the fly without adding noticeable latency.
- **Client Agnostic:** Plays nicely with terminal-based agents like `pi` and `opencode`.
- **Local-First:** Designed with local developer workflows in mind (M1 MacBooks, `llama.cpp`, local endpoints).

## Quick Setup

Check the root [monorepo README](../../README.md) for overall workspace instructions, or install and run directly using your preferred Python package manager (`uv` recommended):

```bash
uv sync
uv run proxytool
```