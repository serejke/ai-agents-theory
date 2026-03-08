# AI Agents Theory

A first-principles formalization of the building blocks of AI agent systems — what bricks tools like Claude Code, Devin, Codex, Cursor, and GitHub Actions are assembled from, and what new systems can be built from them.

## Why This Exists

Every agent system — whether it's a CLI tool, a GitHub bot, or a multi-agent research pipeline — is assembled from the same small set of primitives. But most documentation describes agents in terms of specific frameworks (LangChain, CrewAI, AutoGen) rather than the universal building blocks underneath.

This repository formalizes those building blocks using TypeScript-like pseudocode. The goal: if you understand the primitives, you can design any agent system from scratch — or look at any existing one and immediately see which primitives it uses and which it's missing.

## Reading Order

1. **[Agent Primitives](docs/agent-primitives.md)** — Start here. Layers 0-5: from stateless LLM calls to multi-agent composition. The core taxonomy.
2. **[Agent Patterns](docs/agent-patterns.md)** — Planner, Router, Evaluator, Guardrails, State Machine. Which are true primitives and which are compositions?
3. **[Memory](docs/memory.md)** — The most underdeveloped primitive. Episodic, semantic, procedural memory. Four architectural questions.
4. **[Channel](docs/channel.md)** — How agents pass intermediate results. Six variants: filesystem, return value, artifact, shared state, mailbox, git.
5. **[App Inversion](docs/app-inversion.md)** — How applications decompose in the agent world. What inverts, what stays traditional, and the three-layer model.

## The Primitives Map

```
=== PRIMITIVES (irreducible) ===

LLM              stateless: (system, messages) -> stream
Tool             side effect: (params) -> result
Memory           persistent read+write context across sessions
Guardrail        hard constraint, not controlled by LLM
StateMachine     phases + transitions + human checkpoints
Channel          intermediate results between agents within a run

=== PATTERNS (compositions of primitives) ===

AgentLoop        LLM + Tool[] in a cycle          (LLM + Tool)
Session          AgentLoop + history               (AgentLoop + state)
Planner          Session with read-only tools      (Session + constraint)
Router           dispatch by input                 (function)
Evaluator        LLM-as-judge + feedback loop      (LLM + loop)

=== INFRASTRUCTURE ===

Trigger          what starts it (CLI / issue / cron / webhook)
Output           where it writes (stdout / PR / comment)
Environment      code / runtime / OS
```

All primitives are **universal** — none are specific to any particular tool or framework. Claude Code, Codex, Devin, Gemini CLI, Cursor — different assemblies from the same bricks. The difference is in specific parameter values, not architecture.

## Who This Is For

Engineers who want to build agent systems from first principles rather than learn a framework. If you've used Claude Code, Cursor, or Devin and wondered "what's actually going on under the hood?" — this is for you.

## Status

Living documents. The primitives evolve as new agent systems emerge and are analyzed.

## License

[MIT](LICENSE)
