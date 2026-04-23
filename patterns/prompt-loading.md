# PromptLoading

PromptLoading is the pattern of strategies by which content enters the [Prompt](../primitives/prompt.md). Every piece of knowledge the LLM uses — project conventions, skills, retrieved memory, tool outputs — arrives in the Prompt through some loading strategy: eager at assembly time, lazy on the LLM's request, progressive disclosure, or per-turn dynamic injection.

**Composition**: `source → strategy → Prompt slot`. A source (a file, a function call, a retrieval index, a tool) supplies content; a strategy decides when and how that content enters the Prompt; the Prompt slot (`system`, `messages`, `tools`) is where it lands.

## Why It Matters

The [Prompt](../primitives/prompt.md) is a fixed-size window. Everything going into it competes for the LLM's attention and for context-window capacity. Deciding what loads when — and how — is one of the most consequential design choices in an Agent. Eager-loading everything works until the context window fills up. Lazy-loading scales but requires the LLM to know content exists and to request it. The choice of strategy determines both what the Agent can reason about and how it scales.

This pattern is often mis-named "context injection," which collides with two other meanings of "context": the LLM's input buffer and the active conversation ([Session](../harness/session.md)). PromptLoading is specifically about loading strategies — when content enters the Prompt, not where the content comes from or what it means.

## Formal Definition

```typescript
// A Source produces content that will be placed into a Prompt slot.
// The result is either raw text, or message-shaped entries (for the `messages` slot).
type Source = () => string | Message[];

type LoadingStrategy =
  | "eager" // loaded at Prompt-assembly time, always present
  | "lazy" // agent pulls via a tool/reference when relevant (Skills)
  | "progressive" // minimal upfront, expand on request
  | "dynamic"; // per-turn insert (tool results, retrieved docs)

type PromptLoader = {
  source: Source;
  strategy: LoadingStrategy;
  slot: "system" | "messages" | "tools";
};
```

A specific Agent has one or more Loaders configured. Each Loader names a content source, a strategy for when it fires, and which Prompt slot it writes into.

---

## Strategies

### Eager — loaded at assembly time

```typescript
const claudeMdLoader: PromptLoader = {
  source: () => readFile(".claude/CLAUDE.md"),
  strategy: "eager",
  slot: "system",
};
```

Content is loaded when the Prompt is assembled, before the LLM call. Examples: CLAUDE.md, project structure, git status, agent identity, role instructions.

**Strength**: Guaranteed available. The LLM sees the content from the first turn.
**Weakness**: Costs tokens on every call, whether or not the content is relevant to the current question. Fills context window capacity linearly.

Eager works while eager content fits comfortably. When it stops fitting, switch strategy.

### Lazy — loaded on the LLM's request

```typescript
const pdfSkill: PromptLoader = {
  source: () => readFile("~/.skills/pdf/SKILL.md"),
  strategy: "lazy",
  slot: "messages", // appears as a tool result when invoked
};
```

Only a pointer (name + description) lives in the Prompt eagerly — often as a tool schema or a short skill listing. The full content loads only when the LLM decides it's needed and invokes a reference (e.g., reads the skill file via a `read` tool).

**Strength**: Scales — the Agent can have hundreds of skills without blowing up the Prompt.
**Weakness**: The LLM has to recognize when to pull. If the description is weak or the skill is obscurely named, the content won't be loaded even when it would help.

Claude Skills, pi's Skill convention, and Cursor's indexed docs are all lazy PromptLoading — the filesystem or the index holds the content; the Prompt holds only enough to make the LLM pull.

### Progressive disclosure — minimal upfront, expand on request

A variant of lazy loading where content is structured into layers of increasing detail. The first layer is small and eager; deeper layers load on request.

```typescript
// Skill index (eager, tiny) — names + one-line descriptions
const skillIndex: PromptLoader = {
  source: listSkills,
  strategy: "eager",
  slot: "system",
};

// Full skill content (lazy) — agent reads via `read` tool when needed
const skillContent: PromptLoader = {
  source: readSkill,
  strategy: "lazy",
  slot: "messages",
};
```

Used when the full content is too big to load eagerly but you want the Agent to know something exists. The eager layer is the "table of contents"; the lazy layer is the "chapter."

### Dynamic — per-turn insert

```typescript
const retrieveRelevantMemory: PromptLoader = {
  source: () => memoryStore.recall(currentTask),
  strategy: "dynamic",
  slot: "messages",
};
```

Content is freshly gathered each turn based on what the current question is about. Examples: vector-store retrieval (RAG), tool results appended as the Agent runs, just-in-time file reads based on the LLM's requests.

Dynamic strategies are how [Memory](memory.md) usually enters the Prompt: the memory store is the source, a retrieval function is the strategy, and relevant entries land in `messages` or `system` just before each LLM call.

---

## Picking a Strategy

| Content                            | Strategy           | Why                                                      |
| ---------------------------------- | ------------------ | -------------------------------------------------------- |
| Agent identity, core instructions  | Eager              | Must be present; small; defines behavior                 |
| Project conventions (CLAUDE.md)    | Eager              | Small enough; needed every turn                          |
| Skill library (name + description) | Eager (index only) | Enables the LLM to discover what's available             |
| Skill content                      | Lazy               | Content is long; only relevant when skill is invoked     |
| Tool schemas                       | Eager              | Needed so the LLM knows what it can call                 |
| Tool results                       | Dynamic            | Produced per-turn, injected as they arrive               |
| Retrieved memory                   | Dynamic            | Query is per-turn; freshly relevant each time            |
| File contents                      | Lazy or dynamic    | Too big to preload; LLM requests via `read` or retrieval |

The defaults converge because the trade-offs do: eager for small-and-always-needed, lazy for big-and-sometimes-needed, dynamic for fresh-each-turn.

---

## PromptLoading vs. Memory

[Memory](memory.md) is a content source with persistence semantics (cross-session knowledge). PromptLoading is the strategy by which memory content enters the Prompt — typically dynamic retrieval (the memory store returns entries relevant to the current turn) or eager load (if the memory is small enough to always include, like CLAUDE.md). Memory defines _what_ is stored and _how_ to store/retrieve; PromptLoading defines _when_ the retrieval happens and _where_ the result lands.

---

## Related

- [Prompt](../primitives/prompt.md) — what PromptLoading fills
- [LLM](../primitives/llm.md) — what reads the filled Prompt
- [Memory](memory.md) — a content source that typically reaches the Prompt via dynamic retrieval
- [Session](../harness/session.md) — provides message history (an always-eager ingredient of the Prompt)
- [Workspace](workspace.md) — directory hierarchy as a navigable source for lazy loading (the LLM traverses until it finds what it needs)
