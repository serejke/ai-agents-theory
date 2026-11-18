# PromptLoading

The pattern PromptLoading controls when a text goes into the input of the LLM.
Project rules, skills, facts from memory and tool outputs all go to the LLM
through the [Prompt](../primitives/prompt.md). Each of them uses one method of
PromptLoading: at the start, on request of the LLM, in layers, or new in each
turn.

**Composition**: `source → strategy → Prompt slot`.

The composition has these parts:

- The **source** gives the text. Examples are a file, a function call, an index
  and a tool.
- The **strategy** tells when and how the text goes into the Prompt.
- The **Prompt slot** is the part of the Prompt that gets the text: `system`,
  `messages` or `tools`.

<a id="why-it-matters"></a>

## Value

The [Prompt](../primitives/prompt.md) cannot be larger than the context window.
All its contents use space in the context window, and the LLM must divide its
reasoning between all its contents. Thus the decision about which text loads,
and when, is one of the most important decisions for an Agent.

The strategy controls these properties of an Agent:

- The text that the Agent can use in its reasoning.
- The quantity of text that the Agent can have available.

If you load all the text at the start, this operates until the context window is
full. If the LLM loads the text on request, the Agent can have much more text.
But then the LLM must know that the text is available, and it must call a tool
to get it.

This pattern also has the name "context injection." That name is not correct.
The word context has two other meanings in agent systems: the input of the LLM
and the active conversation ([Session](../harness/session.md)). This pattern is
only about the strategy: when a text goes into the Prompt. It is not about the
source of the text or about its meaning.

<a id="formal-definition"></a>

## Definition

```typescript
// A Source gives the content for one Prompt slot.
// The result is raw text, or messages (for the `messages` slot).
type Source = () => string | Message[];

type LoadingStrategy =
  | "eager" // loaded when the Prompt is built, always present
  | "lazy" // the agent pulls it with a tool or a reference (Skills)
  | "progressive" // a small part first, more on request
  | "dynamic"; // added each turn (tool results, retrieved documents)

type PromptLoader = {
  source: Source;
  strategy: LoadingStrategy;
  slot: "system" | "messages" | "tools";
};
```

An Agent has one or more loaders. Each loader gives these items:

- The source of the text.
- The strategy, which tells when the loader adds the text.
- The Prompt slot that gets the text.

<a id="strategies"></a>

## The strategies

Each strategy has a name in the `LoadingStrategy` type: `eager`, `lazy`,
`progressive` and `dynamic`.

<a id="eager--loaded-at-assembly-time"></a>

### `eager`: load at the start

```typescript
const claudeMdLoader: PromptLoader = {
  source: () => readFile(".claude/CLAUDE.md"),
  strategy: "eager",
  slot: "system",
};
```

The harness loads the text when it builds the Prompt, before the LLM call.
Examples are `CLAUDE.md`, the structure of the project, the `git status`, the
description of the agent and the instructions for its function.

- **Strong point**: the text is always available. The LLM sees it from the first
  turn.
- **Weak point**: the text uses tokens in each call, also when the task of the
  turn does not use it. Each new item fills more of the context window.

Use `eager` while all the `eager` text has sufficient space in the context
window. When the space is not sufficient, use a different strategy.

<a id="lazy--loaded-on-the-llms-request"></a>

### `lazy`: load on request of the LLM

```typescript
const pdfSkill: PromptLoader = {
  source: () => readFile("~/.skills/pdf/SKILL.md"),
  strategy: "lazy",
  slot: "messages", // shows as a tool result when the LLM calls it
};
```

At the start, the Prompt holds only a pointer: a name and a description. This
pointer is usually a tool schema or a short list of skills. The full text loads
only when the LLM selects it. For example, the LLM reads the skill file with a
`read` tool.

- **Strong point**: the Agent can have hundred or more skills, and the Prompt
  stays small.
- **Weak point**: the LLM must know when to pull the text. The description can
  be weak, or the name of the skill can be not clear. Then the LLM does not load
  the text, also when the text can help.

The skills of Claude, the skill format of PI, and the documents in the index of
Cursor all use PromptLoading with the `lazy` strategy. The filesystem or the
index holds the text. The Prompt holds only sufficient text to make the LLM pull
the full text.

<a id="progressive-disclosure--minimal-upfront-expand-on-request"></a>

### `progressive`: load in layers

The `progressive` strategy is a type of the `lazy` strategy. The text has
layers, and each layer gives more information than the layer before it. The
first layer is small and loads at the start. The deeper layers load on request.

```typescript
// Skill index (eager, very small): names and one-line descriptions
const skillIndex: PromptLoader = {
  source: listSkills,
  strategy: "eager",
  slot: "system",
};

// Full skill content (lazy): the agent reads it with the `read` tool
const skillContent: PromptLoader = {
  source: readSkill,
  strategy: "lazy",
  slot: "messages",
};
```

Use this strategy if the full text is too large for the start. With this
strategy, the Agent knows that the text is available. The first layer is the
index. The deeper layers are the full text.

<a id="dynamic--per-turn-insert"></a>

### `dynamic`: load new text in each turn

```typescript
const retrieveRelevantMemory: PromptLoader = {
  source: () => memoryStore.recall(currentTask),
  strategy: "dynamic",
  slot: "messages",
};
```

The harness gets new text in each turn. The task of the turn selects the text.
Examples:

- The documents that a vector store finds (RAG).
- The tool results that the harness adds while the Agent runs.
- The files that the harness reads when the LLM calls the `read` tool.

[Memory](memory.md) usually goes into the Prompt with the `dynamic` strategy.
The memory store is the source, and a function that finds entries is the
strategy. The related entries go into `messages` or `system` immediately before
each LLM call.

<a id="picking-a-strategy"></a>

## Select a strategy

| Text                             | Strategy            | Cause                                                |
| -------------------------------- | ------------------- | ---------------------------------------------------- |
| Description and instructions     | `eager`             | Necessary, small, and it controls the agent          |
| Project rules in `CLAUDE.md`     | `eager`             | Small, and necessary in each turn                    |
| List of skills with descriptions | `eager`, index only | It lets the LLM find the available skills            |
| Text of a skill                  | `lazy`              | Long, and necessary only when the LLM uses the skill |
| Tool schemas                     | `eager`             | The LLM must know the tools that it can call         |
| Tool results                     | `dynamic`           | The tools make them in each turn                     |
| Facts from memory                | `dynamic`           | Each turn has a new query                            |
| Contents of files                | `lazy` or `dynamic` | Too large for the start                              |

For the contents of files, the LLM uses the `read` tool, or a function finds the
related files.

Many systems make the same selections, because the causes are the same:

- `eager` for text that is small and always necessary.
- `lazy` for text that is large and only necessary for some tasks.
- `dynamic` for text that is new in each turn.

<a id="promptloading-vs-memory"></a>

## PromptLoading and Memory

[Memory](memory.md) is a source of text that stays across Sessions. The strategy
of PromptLoading puts the memory text into the Prompt.

- Memory tells which data the system keeps, and how it keeps and finds the data.
- PromptLoading tells when the system finds the data and which Prompt slot gets
  the result.

Memory usually uses one of these strategies:

- `dynamic`: the memory store gives the entries for the task of the turn.
- `eager`: if the memory is small, the harness loads all of it at the start. An
  example is `CLAUDE.md`.

## Related

- [Prompt](../primitives/prompt.md): PromptLoading fills it.
- [LLM](../primitives/llm.md): reads the full Prompt.
- [Memory](memory.md): a source of text that usually goes into the Prompt with
  the `dynamic` strategy.
- [Session](../harness/session.md): gives the history of messages. The history
  is always an `eager` part of the Prompt.
- [Workspace](workspace.md): a tree of folders that the LLM can move through
  until it finds the text. It is a source for the `lazy` strategy.
