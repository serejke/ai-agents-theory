# ContextProvider

A ContextProvider is a function that gathers knowledge and injects it into the [LLM](../primitives/llm.md)'s system prompt or messages. It is how an agent gains awareness of its environment — the project it's working in, the conventions it should follow, the state of the world around it.

**Composition**: `() → string | Message[]` — a pure function that produces context.

## Why It Matters

The [LLM](../primitives/llm.md) only knows what's in its `system` prompt and `messages`. ContextProvider is the mechanism that bridges the gap between "the LLM knows nothing about this project" and "the LLM understands the codebase, the conventions, and the current git state." Without context injection, every agent interaction starts from absolute zero — the agent doesn't know what language the project uses, what testing framework is configured, or what the directory structure looks like.

## Formal Definition

```typescript
// ContextProvider — a function that gathers context
// and turns it into part of the system prompt
type ContextProvider = () => string | Message[];

// Examples:
const claudeMd: ContextProvider = () => readFile(".claude/CLAUDE.md");
const repoStructure: ContextProvider = () => glob("**/*.ts").join("\n");
const gitHistory: ContextProvider = () => exec("git log --oneline -20");

// Composite system prompt from multiple providers
const buildSystem = (...providers: ContextProvider[]): string =>
  providers.map((p) => p()).join("\n\n");
```

CLAUDE.md is precisely a ContextProvider — not "memory" in the strict sense (the agent doesn't automatically write to it between sessions), but a **knowledge injection** into a stateless function. The distinction matters: [Memory](../primitives/memory.md) is a ContextProvider with write capability (`recall` + `remember`). A plain ContextProvider is read-only.

---

## Eager vs. Lazy Context

ContextProviders have two loading strategies with fundamentally different trade-offs:

### Eager — loaded at session start

```typescript
type EagerContext = () => string;
// Examples: CLAUDE.md, git status, project structure
// Always in context window. Costs tokens on every LLM call.
```

Eager context is always available — the agent has it from the first message. But it consumes context window capacity on every LLM call, whether the information is relevant to the current question or not.

### Lazy — loaded on-demand when the agent decides it needs it

```typescript
type LazyContext = {
  name: string;
  description: string; // agent reads this to decide whether to invoke
  load: () => string; // injected only when invoked
};
```

Lazy context scales — the agent pulls domain knowledge (PDF generation guides, API references, coding standards) only when relevant. The agent sees the `name` and `description` as part of its tool/skill definitions and decides whether to load the full content.

**Eager** works while context is small. **Lazy** scales — keep system prompts small, use lazy injection for domain knowledge.

**Example**: Claude Code's Skill tool loads reference guides on-demand rather than embedding them in the system prompt. A skill named "pdf" with description "Generate and manipulate PDF files" costs a few tokens in the tool listing. The full PDF generation guide (thousands of tokens) loads only when the agent decides the current task involves PDFs.

---

## ContextProvider vs. Memory

The boundary between ContextProvider and [Memory](../primitives/memory.md) is write capability:

| Property      | ContextProvider                          | Memory                                |
| ------------- | ---------------------------------------- | ------------------------------------- |
| Read          | Yes — `() → string`                      | Yes — `recall(query) → entries`       |
| Write         | No                                       | Yes — `remember(entries)`             |
| Update source | External (human edits file, git changes) | Agent itself (extracts and persists)  |
| Typical impl  | CLAUDE.md, git status, project structure | Auto-extracted decisions, preferences |

Memory is a ContextProvider that the agent itself can update. This distinction matters architecturally: ContextProviders are simpler to implement and reason about (just read something and inject it), while Memory requires solving extraction, storage, and retrieval problems.

---

## Related

- [LLM](../primitives/llm.md) — the stateless function that ContextProvider feeds knowledge into
- [Memory](../primitives/memory.md) — ContextProvider extended with write capability
- [Session](session.md) — where ContextProviders are assembled into the system prompt
- [Workspace](workspace.md) — directory hierarchy as a navigable context source
