# Aimlessly

> *Allegedly, actually aimless.*

Aimlessly is a **directed (non-acyclic) decision graph executor** — the procedural memory layer for autonomous agents.

Part of the [Christmas Island](https://github.com/christmas-island) agent stack:

| Library | Role | Status |
|---------|------|--------|
| **[Allegedly](https://github.com/christmas-island/allegedly)** | Knowledge graph (declarative memory) | Implemented (v6) |
| **Actually** | Meilisearch wrapper (queryable knowledge surface) | Designed |
| **Aimlessly** | Decision graph executor (procedural memory) | Design phase |
| **Apparition** | UI for all of the above | Named |

## Core Idea

Synthetic tools as decision opcodes. Each step in a workflow is a micro-prompt with fake tools whose names force a deterministic, structured decision. The LLM is the transition function; the tools are the opcodes; the graph is the program.

Key properties:
- **Fresh synthetic conversations per hop** — no context drift, no hallucination compounding
- **Non-acyclic by design** — cycles are features, not bugs, with meta-inspector oversight
- **AI-editable graphs** — workflows learn and improve themselves
- **Inter-workflow composition** — graphs call each other like function calls with a call stack
- **Model-agnostic routing** — cheap local models for simple decisions, escalate to heavy cloud models when needed

## Status

Early design phase. See [`docs/`](docs/) for design documents and assessments.

## License

MIT
