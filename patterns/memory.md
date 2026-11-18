# Memory

Memory is a storage of knowledge that an agent can read and write, and that
stays after the end of a work session. With Memory, an agent does not start from
zero each time: it keeps the knowledge of its previous sessions. This pattern is
a source of text. A method of [PromptLoading](prompt-loading.md) moves this text
into the [Prompt](../primitives/prompt.md), usually with a step that finds the
related entries on each turn.

**Composition**: a set of [Tools](../primitives/tool.md) (`recall` and
`remember`) on a storage that stays across Sessions. The storage keeps the
knowledge that the agent got from its work, not the full data of the work.
Memory is different from a usual Tool because it gives knowledge from previous
Sessions, and because its data stays after the end of a Session.

<a id="why-it-matters"></a>

## Value

Without Memory, each new Session starts from zero. The agent reads the code
again and finds the structure of the code again. It does not know that in the
previous Session you selected not to use Tailwind. It also does not know the
cause of this decision.

This occurs because of these properties of the other parts:

- The [LLM](../primitives/llm.md) keeps no data between calls.
- A [Session](../harness/session.md) gives the LLM a history in one conversation
  (`history: Message[]`). The context window gives a limit to this history:
  approximately 200,000 tokens.
- The history does not stay. When you close the terminal, the agent cannot get
  the history back.

Memory gives the agent knowledge across Sessions. It gets knowledge from the
work, keeps it after the Session, and finds the related knowledge when new work
starts. Without Memory, each conversation with the agent is the first
conversation. With Memory, the agent operates as a worker who knows the project,
not as a new worker.

<a id="formal-definition"></a>

## Definition

```typescript
type Memory = {
  recall: (query: string) => MemoryEntry[]; // which past knowledge is related to the current work?
  remember: (entries: MemoryEntry[]) => void; // get knowledge from the current work and keep it
};

type MemoryEntry = {
  content: string;
  scope: "global" | "project" | "task";
  kind: "episodic" | "semantic" | "procedural";
  timestamp: Date;
  source: string; // the session, issue, or PR that gave this entry
};
```

A usual configuration of Memory and [PromptLoading](prompt-loading.md) has these
steps:

- `remember()` writes to the storage at the end of each Session, or at each step
  of the work.
- On each turn, a method of PromptLoading calls `recall(query)`. It finds the
  related entries and adds them to the [Prompt](../primitives/prompt.md).

<a id="types-of-memory"></a>

## Types of Memory

The memory of an agent has the same types as the memory of a person. Each type
has a different function. The `kind` field of an entry gives its type.

<a id="episodic--what-happened"></a>

### `episodic`: previous work

The `episodic` type keeps the results of previous work: the steps, their results
and the decisions of the user.

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

<a id="semantic--what-i-know"></a>

### `semantic`: facts

The `semantic` type keeps facts about the project, the code, the area of the
work, and the rules that the user wants.

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

<a id="procedural--how-to-do-it"></a>

### `procedural`: methods

The `procedural` type keeps the workflows that the agent found for the tasks of
this project.

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

<a id="architectural-questions"></a>

## Decisions of a memory system

Each memory system has these decisions: when the agent writes, which data it
keeps, how it finds the data, and where the data is.

<a id="1-when-is-it-written"></a>

### When the agent writes

```typescript
type WriteStrategy =
  | "manual" // a person writes CLAUDE.md
  | "explicit" // the agent calls remember() (Claude /remember)
  | "on_session_end" // automatic extraction after each session
  | "continuous"; // the agent writes during the work
```

The usual methods are `manual` (`CLAUDE.md`) and `explicit` (Claude
`/remember`). A person must do the work in the two methods. A person does not
always do the work, and does not always know which facts are important to the
agent. The `on_session_end` method operates without a person. After each
Session, it gets the decisions and the rules of the user from the history
automatically.

<a id="2-what-is-saved"></a>

### Which data the agent keeps

```typescript
type ExtractionStrategy =
  | "raw_conversation" // the full conversation (too much data)
  | "summary" // a short version of the session (loses details)
  | "facts" // facts: "project uses X"
  | "decisions" // decisions and their causes: "chose X because Y"
  | "preferences"; // rules of the person: "doesn't like Z"
```

The `decisions` and the `preferences` have the most value. It is also not easy
to get them from the history. They make the difference between a new worker and
a worker who knows the project. That worker knows which tools the project uses.
The worker also knows the causes of the decisions and the rules that are
important to the user.

<a id="3-how-is-it-retrieved"></a>

### How the agent finds the data

```typescript
type RetrievalStrategy =
  | "always_load" // load all of it (CLAUDE.md)
  | "keyword_search" // grep over the memory
  | "semantic_search" // embedding and vector similarity (RAG)
  | "llm_curated"; // an LLM selects the entries for the current task
```

`always_load` operates while the memory is small, for example a `CLAUDE.md` file
of less than approximately 1000 lines. For a large memory that increases across
Sessions, use `semantic_search` or `llm_curated`. These methods continue to
operate when the quantity of knowledge increases.

One more method is possible when the memory is a tree of folders. The agent
moves through the tree and uses the names of the folders and files to find the
related context. [Workspace](workspace.md) tells about this method.

<a id="4-where-does-it-live"></a>

### Where the data is

```typescript
type StorageBackend =
  | "local_file" // .claude/CLAUDE.md, .claude/memories/
  | "git_repo" // in a commit with the code (with versions)
  | "database" // vector database for semantic search
  | "platform"; // the cloud of the provider (Claude.ai projects)
```

With `git_repo`, the memory is a part of the project. A new developer or a new
agent gets the memory with `git clone`. It is documentation for new workers, and
an agent writes it for agents.

<a id="scopes"></a>

## Levels of Memory

Memory has a `global` level, a `project` level and a `task` level. When the
agent makes its context, it uses the levels together, from the `global` level to
the `task` level.

```typescript
type MemoryScope = {
  global: Memory; // "this person prefers TypeScript strict mode"
  project: Memory; // "in this project coordinates are in [lat, lng] format"
  task: Memory; // "in task #42 we already tried approach X, didn't work"
};

// The levels go into the context one after the other
const buildContextWithMemory = (task: string): string => {
  const globalMem = globalMemory.recall(task); // rules of the person
  const projectMem = projectMemory.recall(task); // knowledge of the project
  const taskMem = taskMemory.recall(task); // history of this task
  return [systemPrompt, ...globalMem, ...projectMem, ...taskMem].join("\n");
};
```

<a id="current-implementations-and-gaps"></a>

## Implementations

| Implementation               | `kind`                   | `WriteStrategy` | `RetrievalStrategy` | `MemoryScope` |
| ---------------------------- | ------------------------ | --------------- | ------------------- | ------------- |
| `CLAUDE.md`                  | `semantic`, `procedural` | `manual`        | `always_load`       | project       |
| Claude `/remember`           | all types                | `explicit`      | `always_load`       | `global`      |
| History of git               | `episodic`               | automatic       | none                | project       |
| GitHub issues, pull requests | `episodic`, decisions    | automatic       | `keyword_search`    | project, task |
| RAG on the code              | `semantic`               | automatic       | `semantic_search`   | project       |
| Projects of Claude.ai        | `semantic`               | `manual`        | `always_load`       | project       |

No implementation in this table does all of these steps automatically:

- At the end of each Session, it gets the decisions and the rules of the user
  (`on_session_end` with `decisions` and `preferences`).
- It keeps them in a format for `semantic_search` in the next Session.

The nearest equivalent is a person who changes `CLAUDE.md` after some Sessions.
That person does the work of `remember()` for the agent.

<a id="claudemd-as-disguised-shared-memory"></a>

## `CLAUDE.md` as the Memory of the project

`CLAUDE.md` is the most basic Memory. The line "No Tailwind, use CSS custom
properties" is `semantic` memory. The line "After changes, run lint && build" is
`procedural` memory. But a person writes this file manually, and its capacity
has a limit. A large `CLAUDE.md` fails, because `always_load` fills the context
window.

The full Memory pattern closes this loop:

- The agent writes to the memory after each Session (`on_session_end` with
  `decisions`).
- The agent selects the knowledge for the next task (`semantic_search` or
  `llm_curated`).

Then `CLAUDE.md` changes with each Session: it becomes a store of knowledge that
increases. Each new agent on the project starts with the knowledge of all
previous Sessions, not from zero.

## Related

- [LLM](../primitives/llm.md) keeps no data between calls. Memory gives
  knowledge that stays.
- [PromptLoading](prompt-loading.md) moves the text of Memory into the Prompt.
  Memory is the source of the text.
- [Session](../harness/session.md) keeps the history of one conversation. Memory
  keeps knowledge across Sessions.
- [Channel](channel.md) moves data in one run. Memory keeps knowledge across
  runs.
- [Continuity](continuity.md) composes Memory with other patterns. Then a task
  can continue across Sessions.
