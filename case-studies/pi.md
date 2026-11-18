# Case Study: PI

PI is a small coding agent by Mario Zechner. Its code is in the `pi-mono`
repository. Most coding agents, for example Claude Code, Cursor and Codex, come
with full workflows and many features in the core. PI comes with a small core.
At each boundary between two concepts of the theory, PI gives an extension
point, where the user can add code.

The web page `shittycodingagent.ai` has this text about PI: "There are many
coding agents, but this one is mine". The text does not tell that the product is
bad. It tells a decision about the structure of PI. A harness for code work must
adapt to the workflow of its user. The workflow must not adapt to the harness.
To get this result, PI does not select the features of the core.

Of the coding agents that persons use, PI has the structure that is nearest to
the words of this book. Its `README` tells: "primitives, not features". Its
extension system uses the same concept boundaries as the theory:

- the tools
- the hooks on tool calls
- the text that extensions add to the context
- the skills that load on request
- the start and the end of a Session.

Thus PI shows the result when an author divides a coding agent into almost the
same concepts as this book. The author of PI did not read the book.

## Summary

The packages of PI agree with the tiers of the theory:

- `pi-ai` is the [LLM](../primitives/llm.md) and the
  [Prompt](../primitives/prompt.md), with one Prompt format for all providers.
- `pi-agent-core` is the [AgentLoop](../harness/agent-loop.md) and the
  [Guardrail](../harness/guardrail.md) harness, with hooks at each concept
  boundary.
- `pi-coding-agent` is the [Agent](../harness/agent.md) harness. It puts one
  [Session](../harness/session.md) behind many frontends.

Extensions, skills, prompt templates and `pi` packages let users give their
compositions to other users. Thus a large set of extensions can become available
around the small core.

The theory divides into concepts the parts that PI shows to the user. PI shows
to the user the concepts that the theory divides. The two agree about the
concepts. Only their function is different: the theory gives the concepts, and
PI is a full example that operates.

The results go in two directions. Each result has a section in this document.

| Source     | Result                                                                                                           |
| ---------- | ---------------------------------------------------------------------------------------------------------------- |
| PI         | [The graph of Sessions is a Workspace](#the-graph-of-sessions-is-a-workspace)                                    |
| PI         | [Each concept boundary is a hook](#each-concept-boundary-is-a-hook)                                              |
| PI         | [Skills load on request](#skills-load-on-request)                                                                |
| PI         | [Extensions operate with a good supply system](#extensions-operate-with-a-good-supply-system)                    |
| PI         | [Steering messages and follow-up messages](#steering-messages-and-follow-up-messages)                            |
| PI         | [One core with many frontends](#one-core-with-many-frontends)                                                    |
| The theory | [Compaction is PromptLoading, not Memory](#compaction-is-promptloading-not-memory)                               |
| The theory | [`AGENTS.md` is not Memory](#agentsmd-is-not-memory)                                                             |
| The theory | [Plan mode, subagents and MCP adapters are patterns](#plan-mode-subagents-and-mcp-adapters-are-patterns)         |
| The theory | [PI does not supply Continuity](#pi-does-not-supply-continuity)                                                  |
| The theory | [The context window stays a limit](#the-context-window-stays-a-limit)                                            |
| The theory | ["No permission popup": Guardrails and sandboxes compose](#no-permission-popup-guardrails-and-sandboxes-compose) |

PI does not supply some patterns: Memory, Continuity, a StateMachine for
workflows with many phases, and a Topology of many agents. The book gives these
patterns for systems that are larger than a tool for code with one agent.

<a id="what-pi-actually-is"></a>

## The packages of PI

The `pi-mono` repository has seven packages. Three of these packages make the
agent framework. The other packages are not a part of the agent:

- an interface in the terminal
- a web interface
- an assistant for Slack
- a tool for GPU pods.

| Package                         | Function                                                                                                                                                                                                                                                                                |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@mariozechner/pi-ai`           | One layer for all LLM providers. It connects to more than 15 providers with one `stream()` API. It is the [LLM](../primitives/llm.md) and the [Prompt](../primitives/prompt.md), which are primitives. It changes one format for messages and tools into the protocol of each provider. |
| `@mariozechner/pi-agent-core`   | An [AgentLoop](../harness/agent-loop.md) that keeps its data between calls. It runs tools, sends events, and has queues for steering messages and follow-up messages. Its `beforeToolCall` and `afterToolCall` hooks are the slot of the [Guardrail](../harness/guardrail.md) harness.  |
| `@mariozechner/pi-coding-agent` | The `pi` CLI. It is the [Agent](../harness/agent.md): a frontend, a [Session](../harness/session.md) and an [Environment](../environment/environment.md). It adds the graph of Sessions, extensions, skills, prompt templates, themes and a tool that installs packages.                |

All of PI in one line:

```
PI (the coding agent) = Sessions (AgentLoop + history, stored as a shared-prefix graph)
                     + Extensions (user-authored Tools, Guardrails, PromptLoading strategies, StateMachine phases, commands, UI)
                     + Skills (lazy PromptLoading, Agent Skills standard)
                     + PromptTemplates (eager PromptLoading, user-invoked)
                     + Compaction (lossy Session→Session transform)
                     + AGENTS.md/CLAUDE.md (eager PromptLoading, always-loaded)
                     + PiPackages (npm/git distribution of the four above)
                     + FourDeployments (interactive, print, JSON, RPC, SDK)
```

Each term after `Sessions` is a surface where the user composes. It is not a
feature of the core. That is all of the structure.

<a id="what-pi-reveals"></a>

## The results of PI for the theory

These results apply to agent systems in general, not only to PI.

<a id="1-the-session-graph-is-navigable-state"></a>

### The graph of Sessions is a Workspace

Each [Session](../harness/session.md) of PI is linear, and PI does not change
the Session harness. PI adds a graph of Sessions that start with the same
prefix. The user can move a pointer through this graph. Each end of the graph is
a linear history. The new part is the relation between the Sessions: the entries
that they have together. The user can see this relation and move through it.

PI keeps the Sessions in a JSON Lines file:

- Each entry has an `id` and a `parentId`.
- One file holds many branches.
- A pointer identifies the active end of a branch.
- `buildSessionContext()` goes from the end of the branch to the root, and makes
  a flat list.

Thus the LLM always sees a linear history. This history is the same as the
`Message[]` of the theory.

The user moves through the graph with these commands:

- `/tree` moves the pointer to the end of a different branch. This changes the
  active Session.
- `/fork` makes a new Session. Its history is a prefix of a different Session,
  until an entry that the user selects.
- `/clone` makes a copy of the active path.
- `branch_summary` records a summary of a path before the user goes away from
  it.
- `label` gives a name to a node. Then the user can find the node again.

Most coding agents do not show the graph of Sessions. In these agents, the
Sessions are not connected. A new branch is a full copy, and the agent loads
only one Session. The user cannot see the entries that Sessions have together.
In PI, these entries are a part of the data, and the graph is the primary part
of the interface:

- All Sessions are in one file.
- To make a new branch, PI writes one entry that knows the pointer.
- To go to a different Session, PI moves the pointer.

This structure is a [Workspace](../patterns/workspace.md) that holds Sessions.
It is not a change to the Session harness. Its nodes are the conditions of the
conversation, and its edges are the links to the previous entries. The graph
helps users who want fast and easy branches:

- to try an alternative
- to keep a path available after they go away from it
- to go back to a previous point.

A workflow that tries different solutions ("try three approaches, keep the one
that worked") gets value from this graph. A usual chat, turn by turn, does not
use it.

<a id="2-every-concept-boundary-should-be-a-user-hook"></a>

### Each concept boundary is a hook

The extension API of PI sends an event at each concept boundary:

| Event                                                  | Concept boundary                                                                          |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| `tool_call`, `tool_result`                             | The Guardrail and the Tool                                                                |
| `context`                                              | The assembly of the Prompt, where [PromptLoading](../patterns/prompt-loading.md) connects |
| `before_provider_request`, `model_select`              | The LLM                                                                                   |
| `session_start`, `session_shutdown`, `session_compact` | The Session                                                                               |
| `resources_discover`                                   | The skills and the prompt templates                                                       |

This table makes the [seams](../verification/seams.md) of the theory into a
contract for extensions: each concept boundary is a seam. If a framework has
clear concepts, it gets its extension API almost free. The API has one hook for
each boundary.

<a id="3-progressive-disclosure-beats-preloaded-capability"></a>

### Skills load on request

In PI, the system prompt holds only the name and the description of each skill.
PI loads the full `SKILL.md` when the model selects the skill. This method uses
a smaller number of tokens than a system prompt with all the skills, and it
accepts more skills. The mechanism is not new: the model reads a file with the
`read` tool.

Agreed rules make the mechanism operate correctly each time:

- the format of Agent Skills
- the `/skill:name` command
- the `disable-model-invocation` field in the header of `SKILL.md`.

Here, agreed rules are more important than new code.

<a id="4-primitives-not-features-is-a-distribution-strategy-not-just-an-architecture"></a>

### Extensions operate with a good supply system

"Primitives, not features" is a decision about the structure of PI. It is also a
decision about the supply of extensions. These properties make the extension
system of PI operate when persons use it:

- PI finds extensions automatically in known folders: `~/.pi/agent/extensions/`
  and `.pi/extensions/`. The user puts a file there, and it runs.
- PI loads TypeScript with jiti. The authors make no build of their extensions.
- Users install extensions with npm or git, and they can lock a version. Thus
  extensions move quickly, at the speed of packages.

Without these properties, a small number of persons write extensions, although
the system can accept them. PI puts the same quantity of work into the supply of
extensions and into the API. Web browsers have WebExtensions and an extension
store. VS Code has an extension marketplace. PI makes the same type of system
for coding agents.

<a id="5-steering-and-follow-up-are-an-out-of-band-user-channel"></a>

### Steering messages and follow-up messages

PI has two channels from the user that operate between the turns. In the Session
model of the book, the user and the agent speak in turns. PI adds:

- A **steering message**: the user pushes `Enter` while tools run. PI gives the
  message to the agent when all tools of this turn have results.
- A **follow-up message**: the user pushes `Alt+Enter`. PI gives the message to
  the agent at the point where the agent stops.

Thus the user does more than speak in turns. The user controls the Session while
it runs, and writes into it at specified points. PI adds this fact to the
Session model: the channel from the user to the agent is different from the
channel back. It also has its specified times.

<a id="6-many-shells-from-one-core-validate-the-agent-harness"></a>

### One core with many frontends

The same `Agent` object of PI operates all the frontends:

- the interface in the terminal
- the mode that prints the result
- the JSON sequence of events
- RPC on `stdin` and `stdout`
- the SDK.

The code of the reasoning has no branches for these frontends. This is the
strongest sign for the [Agent](../harness/agent.md) harness of the book. If the
frontend is a clean boundary, one core is good for each frontend with no change.
PI does this. The section
[Agent: one core, many frontends](#agent-one-core-many-frontends) shows the
frontends.

<a id="what-the-theory-reveals-about-pi"></a>

## The results of the theory for PI

These results go from the book back to the users of PI.

<a id="1-compaction-is-a-promptloading-transform-not-a-memory-write"></a>

### Compaction is PromptLoading, not Memory

PI compaction makes the history shorter in the active Session, and the next
Sessions see no result of it. When the context window is full, compaction writes
a summary of the first messages on the active path of the graph. It replaces a
prefix of these messages with a `CompactionEntry` that holds the summary. The
full history stays in the JSON Lines file, and `/tree` can show it again.
Compaction does not write to a different [Memory](../patterns/memory.md) store.

The theory puts compaction at the seam where the harness assembles the Prompt.
In PI, this seam is the `transformContext` hook. Compaction replaces a prefix of
`Session.history` with a summary. Then PI puts that prefix into
`Prompt.messages`. Thus compaction is a part of
[PromptLoading](../patterns/prompt-loading.md): a method that changes the
Session history for each turn. It does not write to Memory.

If users want to keep the knowledge from compaction across Sessions, they must
add a Memory mechanism. For example, they can change `AGENTS.md`, or write an
extension that writes to disk at the time of compaction. The documents of PI do
not tell this difference clearly. The book tells it.

<a id="2-pi-has-no-memory-layer-agentsmd-is-eager-promptloading"></a>

### `AGENTS.md` is not Memory

Across Sessions, PI keeps only `AGENTS.md`, and the user writes it, not the
agent. This file is not [Memory](../patterns/memory.md) in the meaning of the
theory. It is [PromptLoading](../patterns/prompt-loading.md) at the start: PI
loads a file from the disk into the system prompt for each turn.

PI does not try to give a solution for Memory, and this is a careful decision. A
small number of systems in the field have a solution for Memory, and PI does not
tell that it has one. PI gives the parts for a Memory layer: special entries,
extensions and compaction hooks. Each user can build the Memory layer that they
want on these parts.

The Memory pattern gives a list of items for a PI extension that adds Memory:

- the [types of Memory](../patterns/memory.md#types-of-memory): the events that
  occurred, the facts that the agent knows, and the procedures that the agent
  can do
- the methods that write to Memory and the methods that read from it
- the [levels of Memory](../patterns/memory.md#scopes): the person, the project
  and the task.

For each item, the author of the extension makes a decision. Which type of
knowledge does the extension keep? Who tells the extension to write? Who tells
it to read?

<a id="3-plan-mode-sub-agents-and-mcp-bridges-are-each-a-pattern-composition"></a>

### Plan mode, subagents and MCP adapters are patterns

PI does not put these features in its core. The theory gives a name to each one:

- **Plan mode** is a [StateMachine](../patterns/state-machine.md) with a plan
  phase and a phase that does the plan. Each phase has a different set of tools.
  The extension writes a special entry with the phase. It removes the tools of
  the other phase. It changes the phase on a user command or on an indication
  from the model.
- **Subagents** are a small [Topology](../patterns/topology.md). The first
  Session starts a second [Session](../harness/session.md) with a system prompt
  that comes from the first Session. A [Channel](../patterns/channel.md) sends
  the result back to the first Session: `stdout`, a file or a data area that the
  two Sessions use.
- **MCP adapters** are a [Tool](../primitives/tool.md) adapter. It changes the
  descriptions of an MCP server into PI tools.

Each one is one pattern composition on the harness of PI. PI tells its users
"you can build this". The theory tells which composition you build.

<a id="4-continuity-is-a-cross-session-pattern-pi-doesnt-ship"></a>

### PI does not supply Continuity

PI supplies Sessions, with a graph of Sessions and compaction, but not
[Continuity](../patterns/continuity.md). Continuity is a pattern across
Sessions. It composes Memory, PromptLoading (a start sequence), a Guardrail (a
clean condition) and a Tool (a script that prepares the project). PI supplies
the primitives and the harness for Continuity. It does not supply Continuity.

For a long project on PI, the user builds a Continuity layer:

- an `AGENTS.md` that points to a JSON file with the correct condition of the
  project
- a script that prepares the project at the start
- a Guardrail policy in an extension that keeps the commits clean
- a start sequence in `AGENTS.md`.

PI gives the parts, and the user writes the pattern. This is not an error in the
structure of PI. PI makes this decision with knowledge of its results. PI wants
users to build their workflow, and Continuity is one workflow.

<a id="5-the-session-graph-is-a-partial-solution-to-the-context-window-boundary"></a>

### The context window stays a limit

The book gives a hard limit of the Session: "history is bounded by the LLM's
context window. When it overflows, the session dies or loses the beginning."
These mechanisms of PI make this limit softer:

- **Compaction** keeps less information. Then the history goes into the context
  window. Some data goes out of the Prompt.
- **`/fork`** starts a new Session at a previous point that the user selects. It
  keeps one part of the path and removes the remainder.
- **`branch_summary`** keeps a summary of a path when the user goes to a
  different branch.

The basic limit stays. The LLM sees one Session at a time, and the context
window stays the limit of that Session. The number of Sessions in the graph does
not change this. PI gives structure around the limit, but does not remove the
limit. The theory is correct: Memory across Sessions is the correct solution,
not larger Sessions.

<a id="6-the-no-permission-popup-stance-is-a-guardrail-vs-sandbox-composition-argument"></a>

### "No permission popup": Guardrails and sandboxes compose

PI tells its users to run the agent in a container. Requests for permission must
stay out of the agent. The theory gives this position clearly.

The [Guardrails](../harness/guardrail.md) apply limits at the tool boundary. The
sandboxes apply limits at the Environment boundary. The two layers compose. A
container is a sandbox, at the level of the Environment. A request for
permission is a Guardrail policy, at the level of the Tool.

For code tasks, PI uses a sandbox, and each user selects the Guardrails. Thus
the framework does not make a Guardrail policy necessary for the user. This
position has good arguments. It is also the view of this book: layers that
compose.

<a id="pi-mapped-to-the-theory"></a>

## Each concept of the theory in PI

This section shows how PI makes each concept, one concept at a time.

<a id="llm-and-prompt--provider-unification-as-a-library"></a>

### LLM and Prompt: one library for all providers

`pi-ai` is the [LLM](../primitives/llm.md) and the
[Prompt](../primitives/prompt.md), which are primitives. It gives one format for
messages and tools. This format is a Prompt that does not change with the
provider. Below it, `pi-ai` changes the format into the protocol of each
provider. Examples are Anthropic, OpenAI, Google, Azure, Bedrock, Mistral, Groq,
Cerebras, xAI, OpenRouter, Vercel AI Gateway and more.

The LLM call that follows keeps no data between calls: `Prompt → text stream`. A
user can connect to a provider with a subscription or with an API key. PI
accepts the two methods. The reasoning level is one parameter for all providers
that have it (`off | minimal | low | medium | high | xhigh`).

The theory gives these facts, and `pi-ai` shows each fact in code:

- **The LLM is a function that keeps no data between calls.** The agent loop
  does not know which provider it uses.
- **The model is not the difference between agents.** The same `Agent` object
  connects to each of more than 15 providers with no change to the code.
- **The protocol of the provider is a part of the Prompt.** `pi-ai` keeps the
  protocol of each provider in the layer of the Prompt.

Thus `pi-ai` makes the second fact into the boundary of a library. This operates
only because the Prompt is a concept with a clear boundary. Thus `pi-ai` can
make the Prompt the same for all providers.

<a id="agentloop--a-while-loop-with-extension-seams"></a>

### AgentLoop: a `while` loop with extension seams

`pi-agent-core` makes the [AgentLoop](../harness/agent-loop.md) as a `while`
loop, not as a graph. Its cycle is the same as the cycle of the theory:

```
while true:
    assemble Prompt from (system, messages, tool schemas, inserts)
    stream LLM response, parse tool calls from output text
    if no tool calls → return
    execute tool calls (parallel by default, sequential if any tool opts in)
    append tool results to Session history
```

The important part is the hooks of the loop. Each hook is at a concept boundary
of the theory:

| Hook                                                                  | Concept at the hook                                                                                            |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `transformContext(messages)`, before each LLM call                    | The seam where the harness assembles the Prompt. [PromptLoading](../patterns/prompt-loading.md) connects here. |
| `beforeToolCall({ toolCall, args })` gives `{ block, reason }`        | The [Guardrail](../harness/guardrail.md) hook before the tool runs, at the boundary of tool calls.             |
| `afterToolCall({ toolCall, result, isError })` gives a changed result | The Guardrail hook after the tool runs. It can also record the call.                                           |
| `convertToLlm(messages)` filter                                       | A view of the messages by type. The LLM does not see the special messages of the agent.                        |

This table shows a rule of the book as a runtime API: each concept boundary is a
seam. Extensions subscribe to these events of the loop. At each seam, they can
stop or change the work:

- `tool_call`, `tool_result`
- `context`, `before_provider_request`
- `turn_start`, `turn_end`
- `agent_start`, `agent_end`
- `message_start`, `message_update`, `message_end`.

<a id="session--linear-but-sessions-share-prefixes"></a>

### Session: linear, with the same prefix in many Sessions

Each branch of the PI graph is a linear [Session](../harness/session.md). PI
keeps the Sessions as a graph, where many Sessions start with the same prefix.
The section
[The graph of Sessions is a Workspace](#the-graph-of-sessions-is-a-workspace)
gives the storage, the commands and the tier of this graph.

<a id="tool--bash--read--write--edit-minimal-set-everything-else-by-extension"></a>

### Tool: four default tools, and extensions for the other tools

PI has four default tools: `read`, `write`, `edit` and `bash`. It also has three
optional tools: `grep`, `find` and `ls`. PI has no other tools. It has no MCP,
no subagents, no plan mode and no list of tasks in the core. Each other function
is one of these:

- A **skill**: Markdown and scripts that the agent runs with `bash`.
- An **extension**: TypeScript that calls `pi.registerTool()`.

This is the most basic type of [Tool](../primitives/tool.md). A tool is a
function that changes something out of the LLM. With `bash`, the agent can do
almost all operations. Each tool for one field of work is a composition on top
of these tools.

Thus PI does not use MCP, and this agrees with its structure. MCP is one method
to connect tools. If `bash` and Markdown skills do the same work with less work
to operate them, MCP is optional.

<a id="guardrail--the-tool-dispatch-boundary"></a>

### Guardrail: the boundary of tool calls

PI does not get permission from the user. Its documents tell: "Run in a
container, or build your own confirmation flow with extensions." But the
mechanism is always available: `beforeToolCall` and `afterToolCall` run on each
tool call. This is the [Guardrail](../harness/guardrail.md) harness of the
theory. It is a layer at the boundary of tool calls, and its policy slot is
empty by default.

The text "PI has no guardrails out of the box" tells that PI has no policies. It
does not tell that PI has no hook point. The hooks are a primary API of
`pi-agent-core`. The first example of the documents of PI stops `rm -rf`:

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
    const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

This code is a Guardrail policy at the hook of tool calls. PI does not call it a
Guardrail, but the type signature is the same. The hooks are the slot of the
structure, also when no policies are in it.

<a id="promptloading--laziness-tiers"></a>

### PromptLoading: from "at the start" to "on request"

PI uses the [PromptLoading](../patterns/prompt-loading.md) pattern in different
types. Each type loads its text at a different time:

| Mechanism                               | Time of the load         | Cause of the load                                                                                   |
| --------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------------------- |
| `AGENTS.md` / `CLAUDE.md` / `SYSTEM.md` | At the start, always     | PI loads the file when it starts, and uses it for each turn.                                        |
| Prompt templates (`/name`)              | At the start, on request | The user writes `/templatename`.                                                                    |
| Skills (Agent Skills format)            | On request of the model  | The model sees the description in the system prompt. Then it reads `SKILL.md` with the `read` tool. |
| Extensions                              | Each turn, from code     | An extension adds text with the `context` event or with `CustomMessageEntry` entries.               |

Skills are the most important type. The system prompt holds only the names and
the descriptions of the skills. PI loads the full instructions when the model
selects a skill for the task. This is the PromptLoading method that loads the
text in
[steps](../patterns/prompt-loading.md#progressive-disclosure--minimal-upfront-expand-on-request).
PI makes it with an agreed file structure, not with a framework feature:

- The tools (`read` or `bash`) load the text.
- The filesystem keeps the text.
- The model selects the skill.

PI adds no special code for this.

<a id="statemachine--by-extension-not-by-default"></a>

### StateMachine: an extension, not a default

PI has no plan mode. The `README` tells: "Write plans to files, or build it with
extensions, or install a package." Thus extensions supply the
[StateMachine](../patterns/state-machine.md) pattern. If you want phases with
different tools and rules for the change of phase, you write an extension:

- It composes Guardrail policies and a phase variable.
- It removes the tools that the agent cannot use in the phase.
- It keeps the phase with `pi.appendCustomEntry()`.

LangGraph does this differently, because LangGraph puts the StateMachine in its
core. PI makes the StateMachine a surface for composition, not a primary API of
the framework. This agrees with the theory: most coding agents do not use the
StateMachine pattern. A tool loop with one phase is sufficient for usual tasks.
When a task has phases, the phases are different for each task. A language for
phases that does not change then gives more limits than aid.

<a id="memory--compaction-is-session-local-not-memory"></a>

### Memory: compaction stays in the Session

PI compaction is not [Memory](../patterns/memory.md). It changes only the active
Session. Across Sessions, PI keeps no Memory other than `AGENTS.md`. These
sections give more information:

- [Compaction and Memory](#compaction-is-promptloading-not-memory)
- [`AGENTS.md` and Memory](#agentsmd-is-not-memory)

<a id="channel--single-session-single-channel-implicit"></a>

### Channel: one agent, no Channel

PI is a system with one agent. It has no [Channel](../patterns/channel.md),
because no agents send data to other agents. The `README` tells that subagents
are "not built-in, build with extensions or spawn pi instances via tmux". If you
start two PI processes and send data between them, you select the channel:

- the filesystem
- `stdout`
- a git repository
- a folder for messages.

The framework does not select one for you.

<a id="agent--one-core-many-frontends"></a>

### Agent: one core, many frontends

In the [Agent](../harness/agent.md) harness, the frontend, the Session and the
[Environment](../environment/environment.md) can change around a core of
reasoning that stays the same. PI shows this when persons use it. The same
Session runs behind many frontends, as different Agents:

| Frontend                   | Input                 | Output                               |
| -------------------------- | --------------------- | ------------------------------------ |
| Terminal interface         | Keys on `stdin`       | The interface on `stdout` (`pi-tui`) |
| Print mode (`-p`)          | `argv` and `stdin`    | The last text on `stdout`            |
| JSON mode (`--mode json`)  | `argv`                | JSON lines with events on `stdout`   |
| RPC mode (`--mode rpc`)    | JSON lines on `stdin` | JSON lines on `stdout`               |
| SDK (`createAgentSession`) | A function call       | `AsyncGenerator<Event>`              |

The Session, the AgentLoop, the extensions and the skills are the same for each
frontend. Only the frontend changes. The Agent harness gives this division.

<a id="the-composition-hierarchy"></a>

## The levels of composition in PI

PI shows its levels of composition to the user as a part of the interface:

1. **Primitives and harness**, in `pi-ai` and `pi-agent-core`: the LLM and the
   Prompt (`pi-ai`), the AgentLoop, the Session, the Guardrail hooks and the
   events.
2. **The Agent**, in `pi-coding-agent`: the frontends that contain the Session
   and attach it to an Environment. The terminal interface and the modes for
   printed text, JSON, RPC and the SDK all contain the same Session.
3. **Patterns**, in `pi-coding-agent`: the types of PromptLoading (`AGENTS.md`,
   skills, prompt templates), compaction, and the graph of Sessions (a Workspace
   that holds Sessions).
4. **Compositions by the user**, with extensions, skills, prompt templates,
   themes and `pi` packages:
   - the plan mode (the StateMachine pattern)
   - the subagents (the Topology pattern)
   - the MCP adapters (Tool adapters)
   - the requests for permission (Guardrail policies)
   - automatic git commits that keep the work
   - the protection of paths
   - special interfaces
   - games for the time while the agent operates.

The supply mechanism makes each composition a unit that users can give to other
users. Two examples are `pi install npm:@foo/pi-tools` and
`pi install git:github.com/user/repo`. The unit of supply is a function, not a
full program. A `pi` package has the same function for PI that an extension has
for a web browser. But a `pi` package can connect to each seam:

- the tools
- the Guardrails
- the context
- the commands
- the interface
- the providers that PI can use
- the start and the end of the Session.

<a id="why-this-framework-from-first-principles"></a>

## The causes of the structure of PI

The structure of PI follows from a small set of basic facts about the work of a
coding agent.

<a id="premise-1-the-model-is-commodity-composition-is-the-differentiator"></a>

### The composition, not the model, makes the difference

Two coding agents can use the same model and have a 3× difference on the same
tasks. Then the cause is the tools, the method of the context and the
configuration of the Guardrails. The cause is not the selection of the model. A
framework with a composition in its core gives the decisions of its author. A
framework that shows each surface of composition lets the user make the
decisions.

PI is a framework of the second type. This type is correct only if the product
is in the composition, not in the model. The book gives this fact in the
[section about composition](../README.md#composition-is-the-differentiator) of
the `README`. PI shows the fact with the framework that follows from it.

<a id="premise-2-workflows-are-personal-agents-must-adapt"></a>

### Each user has a different workflow, and the agent adapts

One person who builds software for Solana has different tasks from a group of
developers who build a web program with a database. They want different tools,
Guardrails, context and permissions. An agent product with one workflow is good
for one group of users, and bad for each other group. The alternative is to
supply the core and let each group of users compose an agent for its work. A
supply layer that is almost the same as a package system makes this possible.

PI is for a person who makes a copy of the code of a coding agent to change it.
PI replaces the copy with an extension. This changes how the user thinks about
the work. The user does not change the product of a different person. The user
composes an agent from the same parts that the core supplies.

<a id="premise-3-agent-behavior-is-configuration-and-configuration-should-hot-reload"></a>

### The agent is configuration, and it loads again while it runs

The `/reload` command loads the extensions, skills, prompt templates and themes
again. Thus a long Session can change how the agent operates during a task. The
agent does not start again. This follows when the configuration is code. The
agent is a composition, and the composition is a folder of TypeScript files.
Thus when PI loads the folder again, it loads the agent again.

This is important for the [Continuity](../patterns/continuity.md) pattern.
During a long project, the user gets more knowledge about the correct operation
of the agent. With `/reload`, the user can make the agent better while they use
it, and the Session keeps its data.

<a id="premise-4-the-right-concept-decomposition-makes-features-accidental"></a>

### The correct concepts make features easy

A framework can have hooks at each concept boundary: the primitives, the harness
and the Environment. Then each feature on the lists of the other agents is a
small project:

- the plan mode
- subagents
- MCP
- requests for permission
- lists of tasks
- `bash` processes that continue while the agent operates
- automatic git commits that keep the work
- runs in a sandbox.

You do not supply these features. You supply the surface that makes them easy.
This follows from a rule of the theory: a full set of concepts can make each
feature as a composition. When the primitives, the harness and the Environment
are correct, features come from patterns of these concepts. They do not come
from new code in the core.

PI is a test of this rule. The extensions that operate are a sign that the rule
is correct:

- a Doom extension
- an extension for subagents
- a permission check
- a theme that looks the same as Claude Code.

<a id="why-its-worth-exploring"></a>

### The users who get value from PI

PI is the coding agent for a developer who thinks the same as its structure.
This developer:

- always writes special tools, and writes extensions for each tool that they use
- thinks in systems, and wants to know the division into concepts more than each
  part of the API
- operates in Environments where one permission model for all tasks is incorrect
- thinks that a clear composition is more important than a large number of
  features.

Examples of these Environments are containers, computers that the agent uses
through SSH, live servers with errors and temporary virtual machines.

PI does not make the composition decisions for you. It makes you make them. For
a person who wants to make these decisions, this is a good property. For a
person who wants a full product, PI is the incorrect selection. The `README`
tells this: "pi ships with powerful defaults but skips features like sub agents
and plan mode. Instead, you can ask pi to build what you want or install a third
party pi package that matches your workflow."

## Related

- [LangGraph case study](langgraph.md): a framework that puts all its parts in
  the core. LangGraph selects the composition, and PI does not.
- [AgentLoop](../harness/agent-loop.md): the `while` loop in the core of PI.
- [Session](../harness/session.md): Sessions stay linear, and PI shows a graph
  of Sessions with the same prefixes.
- [Workspace](../patterns/workspace.md): the graph of Sessions is a Workspace
  that holds Sessions.
- The [PromptLoading](../patterns/prompt-loading.md) pattern: PI loads
  `AGENTS.md`, skills and prompt templates at different times.
- [Guardrail](../harness/guardrail.md): the `beforeToolCall` and `afterToolCall`
  hooks of PI.
- [Continuity](../patterns/continuity.md): PI does not supply it. A PI user with
  a long project builds it on top of PI.
- [Agent](../harness/agent.md): one core, and many Agents from the same Session.
  The frontends are the terminal interface and the modes for printed text, JSON,
  RPC and the SDK.
- [Verification](../verification/index.md): each concept boundary is a seam, and
  PI makes each seam an extension point.
