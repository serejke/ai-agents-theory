# AI Agents Theory

Theoretical repository formalizing the building blocks of AI agent systems.

## Structure

- `docs/agent-primitives.md` — Core taxonomy (Layers 0-5)
- `docs/agent-patterns.md` — Planner, Router, Evaluator, Guardrails, StateMachine
- `docs/memory.md` — Memory primitive deep dive
- `docs/channel.md` — Channel primitive and variants
- `docs/app-inversion.md` — How applications decompose in the agent world
- `docs/app-inversion-economics.md` — Economics of agent capabilities (pricing, marketplaces, defensibility)
  - `docs/app-inversion-economics-gpt-store.md` — Case study: GPT Store failure analysis
  - `docs/app-inversion-economics-mcp-registries.md` — Case study: MCP registries and automation platform pivot

## Conventions

- All code examples use TypeScript-like pseudocode (not runnable code)
- Cross-references use relative markdown links: `[Name](file.md)`
- Each document is self-contained but links to related docs
- Primitives are irreducible building blocks; patterns are compositions of primitives
- No framework-specific content — everything is universal
- Notes are living thought, not append-only — when adding new insights, integrate into existing structure (rewrite sections, restructure flow). Never patch with "corrections" or "addendum" sections. Git history preserves previous versions
