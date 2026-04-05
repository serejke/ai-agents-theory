# AI Agents Theory

A first-principles formalization of the building blocks of AI agent systems. The book answers: what bricks are agent systems assembled from, and what can be built from them?

## Structure

```
docs/
├── index.md                        — The taxonomy map, motivation, navigation
├── primitives/
│   ├── llm.md                      — LLM: stateless function
│   ├── tool.md                     — Tool: side effects
│   ├── memory.md                   — Memory: persistent read+write context
│   ├── guardrail.md                — Guardrail: hard constraints outside LLM control
│   ├── state-machine.md            — StateMachine: phases, transitions, human checkpoints
│   └── channel.md                  — Channel: inter-agent data passing (6 variants)
├── patterns/
│   ├── agent-loop.md               — AgentLoop: LLM + Tools in a cycle
│   ├── session.md                  — Session: AgentLoop + history
│   ├── context.md                  — ContextProvider: eager vs lazy knowledge injection
│   ├── planner.md                  — Planner: Session with read-only tools
│   ├── router.md                   — Router: dispatch by input
│   ├── evaluator.md                — Evaluator: LLM-as-judge + feedback loop
│   ├── workspace.md                — Workspace: structured data space for agents
│   └── deployment.md               — Trigger + Session + Output + Environment + Composition
├── app-inversion/
│   ├── architecture.md             — Three-layer model, what inverts and what doesn't
│   ├── economics.md                — Markets, pricing, defensibility
│   ├── case-gpt-store.md           — Case study: GPT Store failure analysis
│   └── case-mcp-registries.md      — Case study: MCP registries + automation pivot
├── verification.md                 — Traces, seams, five verification strategies
└── case-studies/
    └── langgraph.md                — LangGraph decomposed into theory primitives
```

## Book Quality Rules

These rules define "done" for any document and must hold as an invariant across the entire book. Verify these after every edit.

### Taxonomy Invariants

- **Primitives are irreducible.** A concept is a primitive if and only if it cannot be decomposed into other primitives. If you can express X as a composition of existing primitives, X is a pattern.
- **Patterns are compositions.** Every pattern document must state which primitives it composes and how. The composition formula is mandatory (e.g., "AgentLoop = LLM + Tool[] in a recursive cycle").
- **The canonical taxonomy lives in `docs/index.md` only.** No other document may contain a "primitives map" or taxonomy table. Individual documents link back to the index for the full picture.
- **Classification is stable.** If a concept is classified as a primitive, it stays a primitive unless the decomposition argument changes. No document may contradict another's classification. If you reclassify something, update every document that references it.

### Document Format

Every document follows this structure (sections may be omitted if genuinely empty, but the order is fixed):

1. **Title** — `# Name` (just the concept name, no decorators like "deep dive" or "analysis")
2. **Opening definition** — One to three sentences defining what this concept is. Not "the problem" — the definition. A reader who reads only this paragraph knows what the concept is.
3. **Why it matters** — Why this primitive/pattern exists, what breaks without it, what it enables. Motivation through consequence, not through narrative setup.
4. **Formal definition** — TypeScript-like pseudocode type definition. Every primitive and pattern must have one.
5. **Body** — Variants, examples, trade-offs, design implications. Structure varies per document.
6. **Related** — Links to related documents. At the bottom, not the top.

### Writing Rules

- **State conclusions, not deliberations.** Never write "is X a primitive or a pattern?" — state which it is and why. The reasoning process belongs in git history, not in the text.
- **No temporal language.** Never write "the most underdeveloped," "doesn't exist yet," "after our deep dive," "updated classification." The book describes the theory as it stands now. Git history preserves evolution.
- **No "Problem" framing as openers.** Don't open with "The Problem" sections that assume the reader already knows there's a problem. Open with the definition, then explain why it matters.
- **The theory is coherent, not self-challenging.** Never frame new insights as exposing gaps, contradictions, or crises in the theory. New aspects of a concept are part of the concept — they extend and deepen it, not undermine it. Write "Guardrails compose with sandboxes to cover generative tools" not "Guardrails face a crisis with generative tools." Write "Generative tools are the interface through which the agent accesses the Environment" not "Generative tools blur the boundary between Tool and Environment." The theory describes how things work; it does not argue with itself.
- **Compositional framing over competitive framing.** When two primitives interact, describe how they compose — not which one "wins" or "replaces" the other. Write "Guardrails enforce at the tool boundary; sandboxes enforce at the environment boundary — both layers work together" not "The sandbox becomes the true hard constraint, not the Guardrail."
- **Framework-specific examples illustrate, they don't define.** "Claude Code implements this as X" is a valid illustration. But the definition must be universal — it must hold for any agent system, not just the one used as example.
- **No duplication of canonical content.** The taxonomy map appears once (in index.md). The composition formula for a pattern appears once (in that pattern's document). If another document needs to reference it, link — don't copy.
- **Integration, not patching.** When adding new insights, rewrite the relevant section to incorporate them. Never append "Note:", "Update:", "Correction:", or "Addendum:" sections. Git history preserves previous versions.
- **All code examples use TypeScript-like pseudocode** (not runnable code). Annotate with comments where the semantics aren't obvious from the types.

### Cross-Reference Rules

- **Relative markdown links only.** `[Memory](../primitives/memory.md)`, not absolute paths or bare names.
- **Every cross-reference must point to an existing document.** Broken links are a build error.
- **Link on first mention per section.** Link a concept the first time it appears in each major section (H2). Don't link every occurrence.
- **No circular dependency in definitions.** Primitive A's _definition_ must not require reading primitive B, which in turn requires reading A. Definitions are self-contained. Cross-references are for deeper understanding, not for the core definition to make sense.
- **Related section at the bottom** lists documents that provide useful context. Keep it to direct relationships — don't list every document in the book.

### Consistency Checks (verify after every edit)

1. **Index matches filesystem.** Every document in `docs/` is listed in `docs/index.md`. Every entry in `docs/index.md` corresponds to an existing file.
2. **Classification agreement.** If `index.md` lists X as a primitive, then X lives in `primitives/` and X's document calls itself a primitive. Same for patterns.
3. **Composition formulas present.** Every pattern document contains an explicit composition formula naming which primitives/patterns it composes.
4. **No orphan taxonomy claims.** No document contains its own "primitives map," "taxonomy table," or "how X fits the primitives map" section. The map is in `index.md` only.
5. **No temporal language.** Grep for: "missing," "underdeveloped," "don't exist yet," "will appear," "after deep-dive," "updated classification." None should appear.
6. **Cross-references resolve.** Every `[text](path.md)` link points to an existing file.
7. **Opening definitions present.** Every document under `primitives/` and `patterns/` starts with a one-to-three sentence definition immediately after the H1 title.
