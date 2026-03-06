# Agent Primitives

Theoretical repository formalizing the building blocks of AI agent systems.

## Structure

- `docs/agent-primitives.md` — Core taxonomy (Layers 0-5)
- `docs/agent-patterns.md` — Planner, Router, Evaluator, Guardrails, StateMachine
- `docs/memory.md` — Memory primitive deep dive
- `docs/channel.md` — Channel primitive and variants

## Conventions

- All code examples use TypeScript-like pseudocode (not runnable code)
- Cross-references use relative markdown links: `[Name](file.md)`
- Each document is self-contained but links to related docs
- Primitives are irreducible building blocks; patterns are compositions of primitives
- No framework-specific content — everything is universal
