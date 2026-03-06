# Memory Primitive — Deep Dive

The most underdeveloped and most important of the missing primitives in current agent systems.

Related: [Agent Primitives](agent-primitives.md)

---

## The Problem

LLM is stateless (see [Layer 0](agent-primitives.md#layer-0-atomic-primitives)). Session gives it "memory" within a single conversation (`history: Message[]`), but this memory is limited (~200K tokens) and ephemeral — close the terminal, everything's lost. Each new session starts from zero: re-reads code, re-"understands" architecture, doesn't remember that yesterday we decided not to use Tailwind and why.

---

## Memory as a Primitive

A regular `ContextProvider` from our model is read-only (read CLAUDE.md, passed to prompt). Memory is a **ContextProvider with write capability**:

```typescript
type Memory = {
  // Recall: what from past experience is relevant now?
  recall: (query: string) => MemoryEntry[];

  // Remember: extract and save knowledge from current interaction
  remember: (entries: MemoryEntry[]) => void;
};

type MemoryEntry = {
  content: string;
  scope: "global" | "project" | "task";
  kind: "episodic" | "semantic" | "procedural";
  timestamp: Date;
  source: string; // from which session / issue / PR
};
```

---

## Three Types of Memory (by analogy with cognitive science)

### Episodic — "what happened"

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

### Semantic — "what I know" (facts about the project)

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

### Procedural — "how to do it" (learned patterns)

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

## Four Architectural Questions

### 1. When is it written?

```typescript
type WriteStrategy =
  | "manual"          // human writes CLAUDE.md by hand
  | "explicit"        // agent explicitly calls remember() (Claude /remember)
  | "on_session_end"  // automatic extraction after each session
  | "continuous";     // agent writes during work
```

Today: `manual` (CLAUDE.md) + `explicit` (Claude `/remember`). Drawback: depends on the human. Humans forget, get lazy, don't know what's important for the agent.

### 2. What is saved?

```typescript
type ExtractionStrategy =
  | "raw_conversation" // entire dialog (too much, doesn't scale)
  | "summary"          // session compression (loses details)
  | "facts"            // extracted facts: "project uses X"
  | "decisions"        // decisions + rationale: "chose X because Y"
  | "preferences";     // human preferences: "doesn't like Z"
```

The most valuable and hardest: `decisions` + `preferences`. This is exactly what distinguishes a "new contractor" from a "team member".

### 3. How is it retrieved on recall?

```typescript
type RetrievalStrategy =
  | "always_load"      // always load everything (CLAUDE.md today)
  | "keyword_search"   // grep over memory
  | "semantic_search"  // embedding + vector similarity (RAG)
  | "llm_curated";    // LLM decides what's relevant for current task
```

`always_load` works while memory is small (CLAUDE.md < 1000 lines). For real persistent memory you need `semantic_search` or `llm_curated`.

### 4. Where does it live?

```typescript
type StorageBackend =
  | "local_file"  // .claude/CLAUDE.md, .claude/memories/
  | "git_repo"    // committed with code (versioned!)
  | "database"    // vector DB for semantic search
  | "platform";   // provider's cloud (Claude.ai projects)
```

Interesting insight: `git_repo` means memory is part of the project. A new developer (or agent) gets memory along with `git clone`. It's like onboarding documentation, only written by an agent for agents.

---

## Three Levels of Scope

```typescript
type MemoryScope = {
  global: Memory;   // "this person prefers TypeScript strict mode"
  project: Memory;  // "in morocco-trip coordinates are in [lat, lng] format"
  task: Memory;     // "in task #42 we already tried approach X, didn't work"
};
```

They nest during recall:

```typescript
const buildContextWithMemory = (task: string): string => {
  const globalMem = globalMemory.recall(task);   // human preferences
  const projectMem = projectMemory.recall(task); // project knowledge
  const taskMem = taskMemory.recall(task);       // history of this task

  return [systemPrompt, ...globalMem, ...projectMem, ...taskMem].join("\n");
};
```

---

## What Exists Today and Where the Gap Is

| Implementation | Memory Type | Write | Recall | Scope |
|---|---|---|---|---|
| **CLAUDE.md** | semantic + procedural | manual | always_load | project |
| **Claude `/remember`** | any | explicit | always_load (?) | global |
| **Git history** | episodic | automatic | no recall | project |
| **GitHub Issues/PRs** | episodic + decisions | automatic | keyword search | project + task |
| **RAG over codebase** | semantic | automatic | semantic_search | project |
| **Claude.ai Projects** | semantic | manual | always_load | project |

**Main gap**: no system that automatically (`on_session_end`) extracts `decisions` + `preferences` and saves them in a format accessible for semantic recall on the next launch.

Today the closest to this is a human manually updating CLAUDE.md after a series of sessions. The human effectively performs the role of the `remember()` function for the agent.

---

## How Memory Fits the Primitives Map

```
LLM ────────────────── stateless: (system, messages) -> stream
  |
  +-- Tool
  |
  +-- AgentLoop
  |    |
  |    +-- ContextProvider --- read-only: () -> string
  |    |    |
  |    |    +-- Memory ────── read+write: recall(query) + remember(entries)
  |    |         +-- scope: global / project / task
  |    |         +-- kind: episodic / semantic / procedural
  |    |         +-- write: manual / explicit / on_session_end / continuous
  |    |         +-- recall: always_load / semantic_search / llm_curated
  |    |
  |    +-- Session ────── AgentLoop + history (= working memory)
  |         |
  |         +-- on_end: session -> Memory.remember()  <-- this is what's missing
  |
  +-- Composition
```

---

## Provocative Observation

CLAUDE.md is **disguised shared memory**. Write "No Tailwind, use CSS custom properties" — that's semantic memory. Write "After changes, run lint && build" — that's procedural memory. But it's a manual process with limited capacity — a 50-page CLAUDE.md stops working because `always_load` fills the context window.

The real breakthrough is when the loop closes: the agent itself writes to memory after each session (`on_session_end` + `decisions` extraction) and itself chooses what to retrieve from memory for the next task (`semantic_search` or `llm_curated` recall). Then CLAUDE.md transforms from a static file into a living, growing knowledge base — and every new agent connecting to the project starts not from zero, but with the accumulated experience of all previous sessions.

