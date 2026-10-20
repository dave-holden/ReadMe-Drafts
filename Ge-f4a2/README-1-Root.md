# proxytoolpatch

A monorepo containing a lightweight local proxy (`proxytool`) and a stream-healing Python library (`toolpatch`) designed to intercept and repair leaked LLM tool calls in-flight.

## The Backstory

Researching for the best coder LLM for my M1 MacBook Pro (32GB RAM) without melting the fans or freezing the machine, the answer inevitably came back as Qwen—specifically, `Qwen2.5-Coder-14B-Instruct-GGUF` running via `llama.cpp`. 

It’s a brilliant model for local coding, but it had other ideas. 

Every time I connected an AI coding-agent CLI (`pi` or `opencode`) and prompted it to do something straightforward—like "list the contents of the current directory"—it would occasionally choke on the tool-calling syntax, outputting raw tool-call JSON objects directly into the chat stream instead of actually executing the command.

| Without `proxytool` (Tool-Call Mismatch) | With `proxytool` (Working Stream) |
| :--- | :--- |
| ![Opencode Error JSON](apps/proxytool/public/assets/images/opencode-showing-tool-call-mismatch.webp) | ![Opencode Working](apps/proxytool/public/assets/images/opencode-showing-directory-content.webp) |
| ![Pi Error JSON](apps/proxytool/public/assets/images/pi-showing-tool-call-mismatch.webp) | ![Pi Working](apps/proxytool/public/assets/images/pi-showing-directory-content.webp) |

## Monorepo Layout

This repo is split into two complementary parts:

- **[`apps/proxytool/`](apps/proxytool/)**: A proxy application that sits between your coding clients and your local LLM backend, catching and fixing malformed tool streams on the fly.
- **[`packages/toolpatch/`](packages/toolpatch/)**: The core Python library handling the heavy lifting of stream-healing, parsing, and patch logic.

## Getting Started

Head over to the individual module directories for setup, configuration, and usage details:
- [proxytool docs & setup](apps/proxytool/README.md)
- [toolpatch docs & setup](packages/toolpatch/README.md)