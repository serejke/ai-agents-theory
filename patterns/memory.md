# Memory

Memory is a persistent read+write store of extracted knowledge that survives across Sessions. It is what separates an Agent that starts from zero every time from one that accumulates experience over time. Memory is a content source; its content reaches the [Prompt](../primitives/prompt.md) via a [PromptLoading](prompt-loading.md) strategy (usually dynamic retrieval).

**Composition**: [Tool](../primitives/tool.md)-set (`recall`, `remember`) over persistent storage, plus the semantic role of storing _extracted knowledge_ rather than raw artifacts. Distinguished from a generic Tool by its cognitive role (long-term recall across sessions) and by its persistence guarantee (survives Session boundaries).

## Why It Matters

The [LLM](../primitives/llm.md) is stateless. [Session](../harness/session.md) gives it history within a single conversation (`history: Message[]`), but this history is bounded by the context window (~200K tokens) and ephemeral — close the terminal, everything's lost. Each new session starts from zero: re-reads code, re-discovers architecture, doesn't remember that yesterday you decided not to use Tailwind and why.

Memory closes this gap. It extracts knowledge from interactions, persists it across sessions, and retrieves relevant knowledge when starting new work. Without Memory, every agent interaction is a first encounter. With it, the agent behaves less like a new contractor and more like a team member.

## Formal Definition

```typescript
type Memory = {
  recall: (query: string) => MemoryEntry[]; // what from past experience is relevant now?
  remember: (entries: MemoryEntry[]) => void; // extract and save knowledge from current interaction
};

type MemoryEntry = {
  content: string;
  scope: "global" | "project" | "task";
  kind: "episodic" | "semantic" | "procedural";
  timestamp: Date;
  source: string; // which session, issue, or PR this came from
};
```

Memory is a content source with persistence semantics; [PromptLoading](prompt-loading.md) is how that content reaches the [Prompt](../primitives/prompt.md). A typical setup: `remember()` writes to the store at session boundaries (or continuously); `recall(query)` is invoked dynamically per turn by a PromptLoading strategy that retrieves relevant entries and inserts them into the Prompt.

---

## Types of Memory

By analogy with cognitive science, agent memory divides into three kinds that serve different purposes.

### Episodic — "what happened"

Events, outcomes, and experiences from past interactions.

```typescript
// "Last time we changed routing, auth tests broke"
// "User rejected the tabs variant, chose timeline"
// "PR #47 was reverted because it broke the mobile version"
{
  kind: "episodic",
  content: "Attempt to extract Sidebar into a separate package failed —
            too many circular dependencies with MapView",
  source: "session-2026-02-15"
}
```

### Semantic — "what I know"

Facts about the project, the codebase, the domain, and the user's preferences.

```typescript
// "This project uses CSS custom properties, not Tailwind"
// "Coordinates in trip-data.json are in [lat, lng] format (Leaflet), not [lng, lat]"
// "Author prefers explicit types and doesn't like any"
{
  kind: "semantic",
  content: "Project uses raw Leaflet via dynamic import with ssr:false,
            NOT react-leaflet",
  source: "CLAUDE.md + codebase analysis"
}
```

### Procedural — "how to do it"

Learned workflows and patterns for accomplishing tasks in this context.

```typescript
// "After code changes always run npm run lint && npm run build"
// "For new components create CSS file in styles/ with custom properties"
// "When patching trip-data.json use RFC 6902 JSON Patch"
{
  kind: "procedural",
  content: "To add a new day to the route: 1) update trip-data.json,
            2) check coordinates in Leaflet format, 3) add photo
            via Unsplash URL, 4) run build",
  source: "extracted from 3 similar sessions"
}
```

---

## Architectural Questions

The design of any memory system reduces to four decisions.

### 1. When is it written?

```typescript
type WriteStrategy =
  | "manual" // human writes CLAUDE.md by hand
  | "explicit" // agent explicitly calls remember() (Claude /remember)
  | "on_session_end" // automatic extraction after each session
  | "continuous"; // agent writes during work
```

Today the dominant strategies are `manual` (CLAUDE.md) and `explicit` (Claude `/remember`). Both depend on the human — humans forget, get lazy, and don't know what's important for the agent. The gap is `on_session_end`: automatic extraction of decisions and preferences after each session without human intervention.

### 2. What is saved?

```typescript
type ExtractionStrategy =
  | "raw_conversation" // entire dialog (too much, doesn't scale)
  | "summary" // session compression (loses details)
  | "facts" // extracted facts: "project uses X"
  | "decisions" // decisions + rationale: "chose X because Y"
  | "preferences"; // human preferences: "doesn't like Z"
```

The most valuable and hardest to extract: `decisions` + `preferences`. This is exactly what distinguishes a "new contractor" from a "team member" — knowing not just what the project uses, but why specific choices were made and what the human cares about.

### 3. How is it retrieved?

```typescript
type RetrievalStrategy =
  | "always_load" // load everything (CLAUDE.md today)
  | "keyword_search" // grep over memory
  | "semantic_search" // embedding + vector similarity (RAG)
  | "llm_curated"; // LLM decides what's relevant for current task
```

`always_load` works while memory is small (CLAUDE.md under ~1000 lines). For real persistent memory you need `semantic_search` or `llm_curated` — strategies that scale with the volume of accumulated knowledge.

A fifth retrieval strategy — **structural navigation** — emerges when memory is organized hierarchically. The agent traverses a directory tree, using names as semantic cues to find relevant context. See [Workspace](workspace.md) for this approach.

### 4. Where does it live?

```typescript
type StorageBackend =
  | "local_file" // .claude/CLAUDE.md, .claude/memories/
  | "git_repo" // committed with code (versioned!)
  | "database" // vector DB for semantic search
  | "platform"; // provider's cloud (Claude.ai projects)
```

The `git_repo` option is particularly interesting: memory becomes part of the project. A new developer (or agent) gets memory along with `git clone`. It's onboarding documentation written by an agent for agents.

---

## Scopes

Memory operates at three levels that nest during recall:

```typescript
type MemoryScope = {
  global: Memory; // "this person prefers TypeScript strict mode"
  project: Memory; // "in this project coordinates are in [lat, lng] format"
  task: Memory; // "in task #42 we already tried approach X, didn't work"
};

// Scopes nest during context assembly
const buildContextWithMemory = (task: string): string => {
  const globalMem = globalMemory.recall(task); // human preferences
  const projectMem = projectMemory.recall(task); // project knowledge
  const taskMem = taskMemory.recall(task); // history of this task
  return [systemPrompt, ...globalMem, ...projectMem, ...taskMem].join("\n");
};
```

---

## Current Implementations and Gaps

| Implementation         | Memory Type           | Write     | Recall          | Scope          |
| ---------------------- | --------------------- | --------- | --------------- | -------------- |
| **CLAUDE.md**          | semantic + procedural | manual    | always_load     | project        |
| **Claude `/remember`** | any                   | explicit  | always_load     | global         |
| **Git history**        | episodic              | automatic | no recall       | project        |
| **GitHub Issues/PRs**  | episodic + decisions  | automatic | keyword search  | project + task |
| **RAG over codebase**  | semantic              | automatic | semantic_search | project        |
| **Claude.ai Projects** | semantic              | manual    | always_load     | project        |

The main gap: no system that automatically (`on_session_end`) extracts `decisions` + `preferences` and saves them in a format accessible for semantic recall on the next launch. Today the closest approximation is a human manually updating CLAUDE.md after a series of sessions — the human effectively performs the `remember()` function for the agent.

---

## CLAUDE.md as Disguised Shared Memory

CLAUDE.md is Memory in its most basic form. Write "No Tailwind, use CSS custom properties" — that's semantic memory. Write "After changes, run lint && build" — that's procedural memory. But it's a manual process with limited capacity — a large CLAUDE.md stops working because `always_load` fills the context window.

The breakthrough is when the loop closes: the agent itself writes to memory after each session (`on_session_end` + `decisions` extraction) and itself chooses what to retrieve for the next task (`semantic_search` or `llm_curated` recall). Then CLAUDE.md transforms from a static file into a living, growing knowledge base — and every new agent connecting to the project starts not from zero, but with the accumulated experience of all previous sessions.

---

## Related

- [LLM](../primitives/llm.md) — stateless; Memory is what gives persistence on top of statelessness
- [PromptLoading](prompt-loading.md) — strategies for getting Memory content into the Prompt; Memory is the source, PromptLoading is the delivery mechanism
- [Session](../harness/session.md) — within-conversation history; Memory is the cross-session equivalent
- [Channel](channel.md) — passes data within a run; Memory persists knowledge across runs
- [Continuity](continuity.md) — composes Memory with other patterns for multi-session task progress
