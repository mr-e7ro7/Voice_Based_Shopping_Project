# Agentic Workflow

## Basic info
Agentic workflows are systems where, instead of executing hardcoded steps, LLMs dynamically decide what actions to take and how to decompose problems based on the tools and tasks provided to them.

### Core Architecture

Every agentic system has roughly the same skeleton:

```
Perception → Reasoning (LLM) → Action → Observation → [loop]
```

The LLM sits in the middle as the **controller**. It reads context, decides the next action, executes it via a tool, reads the result, and continues until the goal is met or it halts.

### Key Design Concepts

**Tool Use** — Agents act on the world through a defined toolset (web search, code execution, DB queries, APIs). The LLM generates a structured call; the environment executes it and returns an observation.

**Memory** — Four types matter:

- In-context: everything in the current window
- External/RAG: retrieved from a vector store (connects to what you know about RAG)
- Episodic: logs of past runs
- Procedural: baked into weights via fine-tuning

**Planning** — How the agent decomposes goals. Two dominant strategies:

- ReAct (Reason + Act):  interleaved chain-of-thought + tool calls in one loop
- Plan-and-Execute:  generate a full plan first, then execute step-by-step

**Human-in-the-Loop** — You can have human involvement at: every step, only on irreversible actions, or not at all.
