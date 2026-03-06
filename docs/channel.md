# Channel Primitive — How Agents Pass State Between Each Other

How do agents in a multi-agent system share intermediate results? Not within a single session (that's `history: Message[]`), but between separate agents working on different phases of a pipeline.

Related: [Agent Primitives](agent-primitives.md), [Agent Patterns](agent-patterns.md), [Memory](memory.md)

---

## The Problem

In a multi-agent pipeline, Agent A produces something that Agent B needs. How does B get it?

This sounds trivial but it's a design decision with real consequences. The choice of channel determines: can agents run in parallel? Can you retry a failed phase? Can you inspect intermediate state? Can you isolate runs from each other?

---

## Channel as a Primitive

```typescript
// Channel — mechanism for passing intermediate results between agents
type Channel = {
  write: (key: string, value: Artifact) => void;
  read: (key: string | pattern: string) => Artifact[];
};

type Artifact = {
  content: string | Buffer;
  type: "text" | "code" | "image" | "structured";
  metadata?: Record<string, unknown>;
};
```

Why is Channel a **primitive** (irreducible), not a pattern?
- It's not a Tool (tools are actions an agent takes; a channel is infrastructure between agents)
- It's not Memory (memory persists across sessions; channels persist across phases within a run)
- It can't be decomposed into LLM + Tool
- Without it, multi-agent systems can't exist — agents would be isolated

---

## Channel Variants

### 1. Filesystem

The simplest. Agents write files to agreed-upon directories. Other agents read them.

```typescript
const filesystemChannel: Channel = {
  write: (key, value) => writeFile(`files/${phase}/${key}`, value),
  read: (pattern) => glob(pattern).map(readFile),
};
```

**Seen in**: multi-agent research pipelines (e.g., researchers write to `research_notes/`, analyst reads them and writes to `charts/`)

| Property | Value |
|---|---|
| Direction | Unidirectional (writer -> reader) |
| Schema | Convention-based (directory = phase, filename = topic) |
| Persistence | Permanent (survives process crash) |
| Isolation | None by default (old files contaminate new runs) |
| Acknowledgment | None (writer doesn't know if reader got it) |

**Pros**: Dead simple. Human-inspectable. Works with any tool that reads/writes files.
**Cons**: No schema enforcement. No run isolation without manual cleanup. No error detection — reader reads whatever's there, including garbage from a failed writer.

### 2. Return Value (Agent-as-Tool)

The child agent's result is returned to the parent as a tool result. The parent then reformulates it into the next child's prompt.

```typescript
const result = await researcherAgent.send("research quantum computing");
// Parent receives result as ToolResult, decides what to pass to next agent
await dataAnalyst.send(`Analyze this research: ${result}`);
```

**Seen in**: any Agent-as-Tool composition (Claude Code Task tool)

| Property | Value |
|---|---|
| Direction | Unidirectional (child -> parent -> next child) |
| Schema | Implicit (string, interpreted by parent LLM) |
| Persistence | Ephemeral (exists only in parent's context window) |
| Isolation | Natural (each invocation is independent) |
| Acknowledgment | Implicit (parent received the result if execution continued) |

**Pros**: Natural composition. Parent can filter/summarize/rewrite before passing along. No shared state to manage.
**Cons**: Bounded by parent's context window. Parent must stay alive for the full pipeline. Lossy — parent may drop important details when summarizing.

**Key insight**: In Agent-as-Tool systems, the **parent agent IS the channel**. It receives child A's output and formulates child B's input. When a filesystem channel also exists, it's actually a side channel — the primary data flow goes through the parent's context.

### 3. Artifact

A structured, typed output produced by one agent and consumed by another. Unlike raw files, artifacts have metadata (type, schema, provenance).

```typescript
type Artifact = {
  id: string;
  type: "research_note" | "chart" | "data_summary" | "report";
  content: string | Buffer;
  schema?: JSONSchema;  // what the content looks like
  producedBy: string;   // agent ID
  timestamp: Date;
};
```

**Seen in**: Claude.ai artifacts, Cursor composer, Replit agent outputs

| Property | Value |
|---|---|
| Direction | Unidirectional |
| Schema | Typed (has explicit type + optional schema) |
| Persistence | Session-scoped (typically) |
| Isolation | Per-session |
| Acknowledgment | None |

**Pros**: Typed — consumer knows what to expect. Inspectable by human. Can carry metadata.
**Cons**: Requires a registry/store. More ceremony than filesystem.

### 4. Shared State Object

A mutable data structure (dict, database row, document) that multiple agents read and write.

```typescript
type SharedState = {
  get: (key: string) => unknown;
  set: (key: string, value: unknown) => void;
  subscribe?: (key: string, callback: (value: unknown) => void) => void;
};
```

**Seen in**: LangGraph state, CrewAI shared memory, custom orchestrators with a central "blackboard"

| Property | Value |
|---|---|
| Direction | Bidirectional |
| Schema | Depends (typed in LangGraph, untyped in blackboard pattern) |
| Persistence | Run-scoped |
| Isolation | Per-run (if implemented correctly) |
| Acknowledgment | Possible via subscribe |

**Pros**: Flexible. Supports non-linear flows (agent B reads what agent A wrote, modifies it, agent C reads the updated version). Natural for iterative refinement.
**Cons**: Concurrency issues (two agents writing to same key). Harder to debug — state mutations are implicit. Tight coupling between agents.

### 5. Message Queue / Mailbox

Asynchronous, decoupled communication. Agents send messages to named channels; recipients consume when ready.

```typescript
type Mailbox = {
  send: (to: string, message: Message) => void;
  receive: (from?: string) => AsyncGenerator<Message>;
};
```

**Seen in**: actor model systems, some multi-agent frameworks (AutoGen conversations)

| Property | Value |
|---|---|
| Direction | Bidirectional / Multi |
| Schema | Message-level (each message is typed) |
| Persistence | Queue-scoped (consumed on read, or persistent) |
| Isolation | Natural (each queue is independent) |
| Acknowledgment | Yes (can implement ack/nack) |

**Pros**: Decoupled. Agents don't need to know about each other. Supports async, parallel, and dynamic topologies.
**Cons**: Complex. Ordering guarantees matter. Overkill for linear pipelines.

### 6. Git

One agent commits changes; another agent reads the diff or checks out the result.

```typescript
const gitChannel: Channel = {
  write: (key, value) => { writeFile(key, value); gitCommit(`Phase: ${phase}`); },
  read: (pattern) => gitDiff("HEAD~1", pattern),
};
```

**Seen in**: GitHub Action workflows (one job commits, another picks up), Devin (commits between planning and implementation)

| Property | Value |
|---|---|
| Direction | Unidirectional |
| Schema | Untyped (files + commit message convention) |
| Persistence | Permanent + versioned (can rollback!) |
| Isolation | Per-branch |
| Acknowledgment | None |

**Pros**: Versioned — can rollback to previous phase. Branching gives natural isolation. Human-reviewable (PRs as checkpoints).
**Cons**: Overhead of git operations. Not suitable for high-frequency communication.

---

## Dimensions for Choosing a Channel

```typescript
type ChannelProperties = {
  direction: "uni" | "bi" | "broadcast";
  schema: JSONSchema | "convention" | "untyped";
  persistence: "ephemeral" | "session" | "run" | "permanent";
  isolation: "per-run" | "per-session" | "shared";
  acknowledgment: boolean;
  inspectable: boolean;  // can a human see intermediate state?
  rollbackable: boolean; // can you undo a phase's output?
};
```

**Rules of thumb:**
- Linear pipeline, single run? **Filesystem** or **return value**.
- Need human inspection between phases? **Filesystem** or **git**.
- Iterative refinement (agent A and B go back and forth)? **Shared state**.
- Dynamic number of agents, async? **Mailbox**.
- Need rollback? **Git**.
- Need schema enforcement? **Artifacts** or **shared state** with validation.

---

## How Channel Fits the Primitives Map

```
LLM ────────────────── stateless: (system, messages) -> stream
  |
  +-- Tool ──────────── side effect: (params) -> result
  |
  +-- Memory ────────── persistent read+write across sessions
  |
  +-- Channel ───────── intermediate results between agents within a run
  |    +-- filesystem
  |    +-- return value (parent as channel)
  |    +-- artifact
  |    +-- shared state
  |    +-- mailbox
  |    +-- git
  |
  +-- Guardrail ─────── hard constraint, not controlled by LLM
  +-- StateMachine ──── phases + transitions + human checkpoints
```

Channel is distinct from Memory: Memory persists **across sessions** (what did we learn yesterday?). Channel passes state **within a run** (what did the researcher find that the analyst needs?).

