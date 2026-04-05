# AI Agents Theory

A first-principles formalization of the building blocks of AI agent systems — what bricks tools like Claude Code, Devin, Codex, Cursor, and GitHub Actions are assembled from, and what new systems can be built from them.

## Why This Exists

Every agent system — whether it's a CLI tool, a GitHub bot, or a multi-agent research pipeline — is assembled from the same small set of primitives. But most documentation describes agents in terms of specific frameworks (LangChain, CrewAI, AutoGen) rather than the universal building blocks underneath.

This repository formalizes those building blocks. If you understand the primitives, you can design any agent system from scratch — or look at any existing one and immediately see which primitives it uses and which it's missing.

## Start Here

**[docs/index.md](docs/index.md)** — The taxonomy map, motivation, and navigation guide.

## Who This Is For

Engineers who want to think about agent systems from first principles rather than learn a framework. If you've used Claude Code, Cursor, or Devin and wondered "what's actually going on under the hood?" — this gives you the vocabulary to answer that question for any agent system, current or future.

## Status

Living documents. The primitives evolve as new agent systems emerge and are analyzed.

## License

[MIT](LICENSE)
