# Case Study: Hermes Agent

Hermes Agent is a framework for an assistant of one person. The assistant
operates all the time. Nous Research makes it.

You install Hermes with `pip install`. Then you can use the same agent from a
terminal or from a chat platform on your phone. A cron task, a webhook or your
IDE can also start it. The agent keeps its history, its memory and its skills
across all these frontends.

One Python class, `AIAgent`, runs the agent for each frontend. Each frontend
uses the same SQLite database of Sessions and the same history with full-text
search. Each frontend also uses the same plugins, memory and skills. Nous
Research also uses this class to run the agent during the reinforcement learning
of the Hermes models. Thus each good change to the harness also helps the
training of the model.

This case study shows how the parts of Hermes agree with the concepts of the
theory. Hermes adds no new concept to the theory. It shows in more depth how
some concepts can operate in a system that persons use.

## Results

These are the important results of the case study. The section
[Concepts in depth](#concepts-in-depth) gives more information about each
result.

- **The system prompt stays the same for all of a Session.** Thus the cache of
  the LLM provider continues to operate. See
  [the section](#the-system-prompt-does-not-change).
- **The agent reads its stored memory in layers.** A snapshot at the start of a
  Session, the tool results in the history, and the files on disk each give a
  view. See [the section](#the-agent-reads-its-memory-in-layers).
- **A different run of the agent writes the memory and the skills.** The agent
  that replies to the user does not write them. See
  [the section](#a-different-run-writes-the-memory).
- **In training, the reward function examines the Environment of the agent.** It
  reads the files that the agent wrote, not a copy. See
  [the section](#the-reward-function-uses-the-same-environment).
- **A profile isolates two agents, not a process.** Many agents can operate in
  one Python process. See [the section](#a-profile-isolates-the-agents).
- **Each frontend runs the same Agent class.** Parameters, not different
  classes, make the differences. See
  [the section](#each-frontend-runs-the-same-class).
- **A trajectory contains only the canonical history.** The text that Hermes
  adds for one call does not go into it. See
  [the section](#a-trajectory-contains-only-the-canonical-history).
- **Memory providers are plugins with one interface.** Hermes lets you use the
  internal provider and a maximum of one external provider. See
  [the section](#memory-providers).
- **The agent can add skills while it runs.** The model does not change, but the
  agent can do more tasks. See [the section](#skills-that-the-agent-writes).

<a id="hermes-vocabulary"></a>

## Terms of Hermes

These terms of Hermes occur in all the case study.

- **`AIAgent`**: the Python class that runs the agent for each frontend. The
  reinforcement learning also runs it.
- **`HERMES_HOME`**: the home folder of a profile. It contains the Sessions, the
  skills, the memory files, the trajectories and the logs.
- **profile**: a configuration with a name. Each profile has a different
  `HERMES_HOME`. The profile isolates two `AIAgent` objects that operate at the
  same time.
- **frontends**: Hermes calls them "trigger surfaces". Each frontend makes an
  `AIAgent` object in the same process. The frontends are:
  - the CLI
  - the Gateway, with an adapter for each chat platform, for example Telegram,
    Discord and Slack
  - cron tasks and webhooks
  - ACP, the Agent Client Protocol for IDEs
  - the reinforcement learning.
- **`SOUL.md`, `MEMORY.md` and `USER.md`**: files with a length limit. Hermes
  loads them into the system prompt at the start of a Session. `SOUL.md` gives
  the persona of the agent. `MEMORY.md` contains the memory entries that the
  agent selects. `USER.md` contains the entries about the user.
- **`SKILL.md`**: the file of a skill. A skill is a folder with a Markdown file
  and optional scripts. The Markdown file starts with a header of metadata. A
  skill gives a procedure that the agent or the user can start again.
- **slash commands**: commands, for example `/skills install` and
  `/memory edit`. If a command changes the data, the change starts in the next
  Session by default. The `--now` flag applies the change immediately.
- **`SessionDB`**: the SQLite database of the Sessions. It has full-text search,
  and it can also find text with trigrams. With trigrams, it also finds words in
  the languages of China, Japan and Korea. A compression of the context starts a
  new Session. A link connects the new Session to the previous Session.
- **trajectory**: a JSONL file in `HERMES_HOME`. Hermes records trajectories
  only if the user selects this option.
- **Honcho**: one of the optional external memory providers. It makes a model of
  the user with a "dialectic" method.
- **Atropos**: the framework of Nous Research for reinforcement learning. It
  runs the rollouts of Hermes and their reward functions.
- **Tinker**: the post-training service of Thinking Machines Lab. It uses
  trajectories and the numbers of their reward functions to change the weights
  of the model.
- **the tools that change the stored data**: `memory`, `skill_manage`,
  `delegate_task`, `session_search` and `todo`. The agent can call them.

## Concepts in depth

Each result gives more depth to a concept that the theory has. Each of the
subsequent sections starts with the name of that concept. Other case studies and
concept documents can use these results.

<a id="prompt-cache-integrity-as-a-runtime-invariant"></a>

### The system prompt does not change

_Concept: [Prompt](../primitives/prompt.md). It also applies to the
[Guardrail](../harness/guardrail.md)._

Hermes keeps the system prompt the same, with no change to one byte, from the
first call of a Session to the last call. The LLM provider keeps the start of
each Prompt in a cache. If the start does not change, the provider can use the
cache for each call. Hermes makes this rule an invariant of the runtime, and its
structure follows this rule. The rule is not a small change for speed that
Hermes adds at the end.

Hermes makes the system prompt one time, at the start of the Session. The system
prompt contains:

- the persona from `SOUL.md`
- the context of the project, which Hermes finds in the working folder
- frozen snapshots of `MEMORY.md` and `USER.md`
- the index of the tools
- the index of the skills.

Each text that changes from turn to turn goes into the last user message. Hermes
adds the text to a copy of that message for one API call only. The copy does not
go into the canonical history. These are examples of this text:

- the results of the memory prefetch
- the context from plugins
- the skills that a slash command starts.

If a slash command changes the text of the system prompt, the change starts in
the next Session by default. Hermes calls this "deferred invalidation". Thus the
agent cannot load system data again during a Session. But the start of the
Prompt stays the same for all of a long conversation, and the cache continues to
operate.

The Prompt has its insert positions only where this rule lets them be. The
Guardrail layer makes sure that Hermes obeys the rule, but not as a policy for
each call. The rule controls the shape of the code: it tells where an insert can
go.

<a id="two-views-onto-durable-harness-state"></a>

### The agent reads its memory in layers

_Concept: [Memory](../patterns/memory.md). It also applies to
[Continuity](../patterns/continuity.md)._

Hermes keeps data on disk: the `MEMORY.md` and `USER.md` files with their length
limit, and the `SKILL.md` folders. The agent reads this data in three views:

- **At the start of a Session**: the agent reads a frozen snapshot in the system
  prompt.
- **During the Session**: the agent reads its tool calls and tool results in the
  history. Each response of `memory` or `skill_manage` contains the list of
  entries after the change. Thus the history always shows the live data.
- **On disk**: the files keep the data and give it to the next Session.

The three views operate together. Only a compression of the context makes Hermes
load the snapshot again during a Session. The compression removes the tool
results that contain the live data. Thus the snapshot must replace them.

Thus Memory in Hermes does not have one method to read the data. It has layers:
the start, the runtime and the storage. A specified condition makes each layer
new again.

<a id="writer-and-responder-as-separate-agent-runs"></a>

### A different run writes the memory

_Concept: [Memory](../patterns/memory.md). It also applies to the
[Evaluator](../patterns/evaluator.md) and
[Continuity](../patterns/continuity.md)._

Hermes does not add "remember to save important facts" to the system prompt of
the agent that replies to the user. After some turns, Hermes starts a different
run of the same `AIAgent` class in a new Session. This run is the reviewer. Its
Prompt is "review this transcript and decide what to persist", and it has the
`memory` tool.

Thus the writer and the agent that replies are different runs with different
Prompts. The agent that replies does only the task of the user. The reviewer
only writes the memory.

The same shape applies to skills. After a task, a reviewer can call
`skill_manage` to make a new `SKILL.md` from the history of the task. The
reviewer does not change the turn of the user or its system prompt.

Thus, in Hermes, a subagent with the shape of an Evaluator writes the memory. An
instruction in the system prompt does not do it.

<a id="the-evaluator-shares-the-rollouts-environment"></a>

### The reward function uses the same Environment

_Concept: [Environment](../environment/environment.md). It also applies to the
[Evaluator](../patterns/evaluator.md)._

In a rollout, the reward function of Atropos and the agent use the same
Environment. The `task_id` identifies this Environment. The test command of the
reward function runs on the files that the model wrote during the rollout, not
on a new copy.

Thus the agent uses the same Environment in training and in its work for users.
It also uses the same `AIAgent` class in the two conditions. Thus Hermes removes
the difference between training and the work for users at the Environment tier
and at the harness tier.

The task, not the run, identifies the Environment. Thus the rollout and its
reward function use the same Environment without special work.

<a id="configuration-scope-as-an-isolation-axis"></a>

### A profile isolates the agents

_Concept: [Topology](../patterns/topology.md)._

A profile, not a process, isolates two `AIAgent` objects that do not use the
same data. A profile sets `HERMES_HOME`. Each part that reads this path uses
only the data of the active profile: the Sessions, the skills, the memory, the
trajectories and the logs. Thus many `AIAgent` objects with different profiles
can operate at the same time in one Python process.

This structure has a problem and a good result:

- **The problem**: plugins and memory providers must be safe for two calls at
  the same time.
- **The good result**: tools, plugins, memory and skills are available on all
  frontends by default. For example, a skill that you install for the CLI is
  also available to a cron task and to the ACP Session.

Thus this Topology isolates agents with the configuration, not with the process.

<a id="one-agent-class-across-every-trigger-surface"></a>

### Each frontend runs the same class

_Concept: [Agent](../harness/agent.md). It also applies to the
[Topology](../patterns/topology.md)._

Each frontend runs the same `AIAgent` class with different parameters: the CLI,
the Gateway with its platforms, cron, webhooks, ACP and the rollouts of
training. A cron task does not start a new process. A webhook makes an internal
message. The Atropos environment runs the same loop.

Parameter flags, not different classes, make the differences between the
frontends. This structure has a problem and a good result:

- **The problem**: the class has many constructor parameters.
- **The good result**: each good change to the harness goes into each frontend
  immediately. This includes the agent in the post-training with Tinker.

The Agent concept tells that one unit that can run connects the Frontend, the
Session and the Environment. Hermes shows this structure without a branch for
each frontend.

<a id="persisted-record-is-canonical-not-per-call"></a>

### A trajectory contains only the canonical history

_Concept: [Prompt](../primitives/prompt.md). It also applies to
[Continuity](../patterns/continuity.md)._

Hermes records trajectories only if the user selects this option, and only in
`HERMES_HOME`. A trajectory contains the canonical history:

- the messages of the user, with the words of the user
- the responses of the model
- the tool calls and the tool results.

A trajectory does not contain the text that Hermes adds for one turn. The
results of the memory prefetch, the context from plugins and the text of the
skills stay only in the copy for one API call.

Thus, when Hermes replays a trajectory, the inputs of the canonical history are
always the same. Hermes does the prefetch again on the live memory data. The
structure keeps the insert positions of the Prompt and the storage of Continuity
on different sides of one boundary.

<a id="memory-pluggable-providers"></a>

## Memory providers

Hermes gives memory as a plugin system with one interface. The internal provider
always loads: the `MEMORY.md` and `USER.md` files. You can also select a maximum
of one of these external providers:

- Honcho
- Mem0
- Supermemory
- ByteRover
- Hindsight
- Holographic
- OpenViking
- RetainDB.

Hermes makes sure of the rule of one external provider when it loads the
providers. The rule prevents the problems of two or more providers:

- The context of one provider and the context of a different provider do not
  agree.
- The providers write the same data two or more times.
- The rules that select the related facts do not agree.

### The steps of a provider

The `AIAgent` loop calls each provider in the same steps:

- `prefetch_all(query)` runs at the start of each turn. Its input is the first
  message of the user. Its result is a context string. Hermes adds the string to
  the user message of this call, not to the system prompt. Thus the start of the
  Prompt does not change. The call does not stop the turn, and a slow provider
  does not make the turn slow.
- `sync_turn(user, assistant)` runs at the end of each turn. It writes the data
  to permanent storage, and the turn does not wait for it. If a provider sends
  an HTTP request to a service on the network, a different thread sends it.
- `queue_prefetch_all` runs after `sync_turn`. It starts the prefetch of the
  next turn, and the agent does not wait for it. Thus the result is available
  when the user replies.
- Hermes calls `on_memory_write` when the agent writes with the `memory` tool.
  Thus an external provider can write the same data as the internal provider.

An external provider can also add tools to the tools of the agent. These tools
give the operations of the provider directly to the LLM. The configuration of
the providers is in `HERMES_HOME`. Thus a profile selects its memory providers
together with all its other configuration.

### Types of providers

The providers keep memory with very different methods:

- **Files**: the internal `MEMORY.md` and `USER.md`. They have a length limit,
  and each write is atomic. Hermes adds them to the system prompt at the start
  of the Session. The agent changes them with the `memory` tool.
- **Vector recall**: the provider finds chunks of the turns of the history with
  the nearest meaning to the query. At the prefetch, it gives the chunks that
  agree best.
- **Knowledge graph**: a store of items and their relations. It gives a view
  with types of the facts about the user.
- **LLM as memory**: Honcho does a "dialectic" LLM call through the network. The
  LLM call uses the model of the user that Honcho keeps, the "peer
  representation". You can set the depth of the call from 1 to 3. The call has a
  timeout of 3 seconds. The next turn waits for the call and gets each result
  that is available. After some turns, Honcho makes its model of the user new
  again, and the agent does not wait for it.

The interface contains structures that are not compatible. Files and an LLM as
memory keep data differently. They also have a different latency and a different
cost. But the interface is small. Thus the agent loop can operate without the
name of the active provider.

The rule of one external provider keeps this interface the same for each
provider. Each provider is the only external source of facts. Thus the steps of
the provider do not select between the facts of different providers.

<a id="self-improvement-runtime-skill-creation"></a>

## Skills that the agent writes

Hermes lets the agent add new procedures while it runs: the agent writes
`SKILL.md` files. This is a change at the harness tier. The model does not
change, but the agent can do more tasks.

A skill is a folder in `HERMES_HOME` with a Markdown file and optional scripts.
The Markdown file starts with a header of metadata. When the skill is on disk,
the agent or the user can start it with these methods:

- as a `/<skill-name>` slash command
- as a cron `--skill` reference
- as a parameter of `delegate_task`.

### How the agent writes a skill

The agent calls `skill_manage` with one of these operations: `create`, `update`,
`view`, `list` or `delete`. On `create`, Hermes does these steps:

1. It makes sure that the header of metadata is correct.
2. It does a security scan of the skill. The next Sessions add the skill to
   their Prompts. Thus a skill must not contain a prompt injection or data for
   exfiltration.
3. It writes `<HERMES_HOME>/skills/<name>/SKILL.md` as one atomic operation.

The skill then stays across Sessions and across frontends. For example, a skill
from a CLI Session is available to a cron task and to an ACP Session in the same
profile.

### One run writes the skill, and a different run uses it

Hermes keeps the work that writes a skill apart from the work that uses it:

- **The reviewer writes the skill.** This is the same division as for memory, in
  [A different run writes the memory](#a-different-run-writes-the-memory). After
  a task that is not easy, the reviewer reads the history of the last turns. It
  makes a decision: can an agent use this procedure again for different tasks?
  If yes, the reviewer calls `skill_manage(action='create', ...)`. The reviewer
  runs in a new Session with a different Prompt.
- **The agent that replies to the user starts the skill.** The user can also
  start it with a slash command. By default, a new skill becomes available in
  the next Session. Hermes loads the skill index of the system prompt again only
  at the start of a Session. If you must have the skill immediately, the `--now`
  flag makes it available.

This division has these results:

- **The agent that writes a skill is not the agent that uses it.** Thus an agent
  cannot "use a skill it just made up" during a task.
- **The start of the Prompt stays the same.** A new skill goes into the system
  prompt only at the start of the next Session.
- **The security scan and the header check are on the write path.** The next
  Sessions add each skill to their Prompts. Without the scan, a skill can be a
  stored prompt injection.

### Skills change when they start

A skill can change when the agent starts it, not only when the reviewer writes
it. Hermes replaces `${HERMES_SKILL_DIR}` and `${HERMES_SESSION_ID}` in the
skill. It can also run short shell commands in the skill. Thus a skill can show
the files of the folder, include the output of `git status`, or add live data to
its instructions.

### Two loops that make the agent better

The skills make the agent better at the harness tier. A slower loop makes the
model better. Nous Research can replay trajectories from the work for users and
give each trajectory a number with the reward functions of Atropos. Then Tinker
uses these trajectories to change the weights of the model for the next Hermes
release.

The two loops use the same `AIAgent` class:

- The skills change the procedures of the agent. They do not change the weights.
- The post-training changes the weights. It does not change the procedures of
  the agent.

In each loop, an Evaluator with the shape of a reviewer selects the results that
stay.

<a id="hermes-mapped-to-the-theory"></a>

## Hermes and the theory

This section gives the part of Hermes for each concept of the theory.

### Primitives

**[LLM](../primitives/llm.md): an adapter for each provider.** One interface
contains an adapter for each of these providers:

- Anthropic
- the Codex Responses API
- OpenAI Chat Completions
- Bedrock Converse
- the API of Gemini
- Gemini CloudCode
- an endpoint of Nous Research.

To change the model, you change the configuration, not the code. Hermes can
replace each LLM with a different LLM.

**[Tool](../primitives/tool.md): two levels of tool calls.** These tools change
the stored data: `memory`, `skill_manage`, `delegate_task`, `session_search` and
`todo`. Hermes stops their calls in the process, at the `AIAgent` level. Thus
these tools can read and write that data directly.

All the other tools go through a registry, with hooks before and after each
call. This division keeps the changes of the data near the internal data of the
agent. The other tools all operate with the same method.

**[Prompt](../primitives/prompt.md): a system prompt that does not change.**
Hermes makes the system prompt one time for each Session. It keeps the system
prompt the same for each API call of that Session. Each text that changes for
each turn goes into a copy of the last user message. That copy does not go into
the canonical history. See
[The system prompt does not change](#the-system-prompt-does-not-change).

### Harness

**[AgentLoop](../harness/agent-loop.md): a loop with a limit that subagents can
use together.** One loop runs the steps in sequence. These conditions stop it:

- a maximum number of cycles.
- a limit of cycles that a subagent can use together with the agent that started
  it. A subagent can also get a different copy of the limit.
- a flag that gives the model one last call after the limit.

A different flag stops the loop if a new user message comes during a run.

**[Session](../harness/session.md): a stored Session and a runtime Session.**
Hermes has two types of Session:

- **The stored Session** is the history in `SessionDB`. A compression stops the
  stored Session and starts a new stored Session. A link connects the two stored
  Sessions. The runtime Session continues. Hermes finds previous history in the
  stored Sessions.
- **The runtime Session** is a live `AIAgent` object. Its key is
  `(platform, chat_id, thread_id)`. It has a message queue, and it runs control
  commands in the Session. Each runtime Session operates at the same time as the
  other runtime Sessions.

**[Guardrail](../harness/guardrail.md): layers at different positions.** Hermes
has these guardrails:

- a list of permitted commands and an approval step before the shell tools
- a scan of each cron Prompt before Hermes adds the cron task. The scan finds
  prompt injections with regular expressions. It also finds Unicode characters
  that a person cannot see.
- check functions for each tool, for the function of that tool. Hermes adds them
  when it adds the tool to the registry.
- hooks of plugins before each tool call. A hook can stop the call.
- the cache rule: no change to the context during a conversation. The positions
  of the insert points and the code inspections of the developers make sure of
  this rule. The rule operates as a guardrail of the structure.

**[Agent](../harness/agent.md): one `AIAgent` class for each frontend.**
`AIAgent` connects the Frontend, the Session and the Environment through
parameters, functions that it calls back, and calls in the same process. Each of
these is an object of the same class with different parameters:

- a turn of work for a user
- a rollout
- the reviewer
- an ACP Session
- a cron task
- a webhook handler
- a subagent that gets a task.

See [Each frontend runs the same class](#each-frontend-runs-the-same-class).

### Environment

**[Environment](../environment/environment.md): backends that you can change.**
The configuration selects the backend: the local computer, docker, SSH, Daytona,
Modal or Singularity. The terminal tool runs its commands in the active backend.
In a rollout, the reward function of Atropos uses the same backend. See
[this result](#the-reward-function-uses-the-same-environment).

### Patterns

**The [PromptLoading](../patterns/prompt-loading.md) pattern: the skills, the
persona and the context of the project.** Each skill is a folder on disk with a
`SKILL.md` file, optional scripts and optional files that it fills with data.
Hermes loads the persona from `SOUL.md`. It finds the context of the project in
the working folder.

Hermes adds each skill that the agent starts as a user message, not as a part of
the system prompt. Thus the start of the Prompt does not change. If a slash
command changes the data of the Prompt, the change starts in the next Session by
default.

**[Memory](../patterns/memory.md): providers that you can change.** Hermes has
the internal files and a maximum of one external provider. Each turn starts a
prefetch, and the turn does not wait for it. At the end of the turn, Hermes
writes to permanent storage and does not wait for it. Then Hermes starts the
prefetch of the next turn.

A different run of the agent writes the memory. See
[Memory providers](#memory-providers) and
[A different run writes the memory](#a-different-run-writes-the-memory).

**[StateMachine](../patterns/state-machine.md): the data of each turn, and cron
tasks.** The loop keeps these data for each turn:

- the number of cycles
- the limit of cycles
- the flag for the last call
- the flag that stops the loop
- the last checkpoint.

Each cron task is a state machine in a file. It can stop for a time, continue,
run immediately or get new data. Hermes locks the file. Thus two or more Gateway
processes can operate at the same time without errors.

**[Channel](../patterns/channel.md): a Gateway for many chat platforms.** The
Gateway receives messages from all the platforms and sends messages to all of
them. Each platform has an adapter with these properties:

- a rate limit
- a rate for the changes to a message after Hermes sends it
- a selection of the types of data that the platform accepts.

Hermes sends the output in small parts, at a rate for each channel. Hermes also
sends messages about the processes that continue to run after their tool call.
Each channel has a control that sets the quantity of text in these messages.

**[Planner](../patterns/planner.md): no special Planner part.** The
`delegate_task` tool gives a task to a different agent. It does not make a plan.
The `todo` tool keeps a short list of tasks with a length limit, not a tree of
plans. The plan of the agent is the output of its reasoning.

**[Router](../patterns/router.md): many small routers, but no router for all the
inputs.** Hermes has these routers:

- a router of LLM providers, in the adapter layer of the LLM
- a router of the backends that make images
- a router of memory providers, in the `MemoryManager`.

**[Evaluator](../patterns/evaluator.md): evaluators at three positions.**

- The reward functions of Atropos are evaluators in the full meaning of the
  word.
- The reviewer is an evaluator of the history of the last turns. It selects the
  memory and the skills to keep.
- Hermes connects to benchmark harnesses, for example terminal-bench and
  mini-SWE. They examine the agent on a large set of benchmark tasks.

**[Workspace](../patterns/workspace.md): the working folder, a folder for each
skill, and the sandbox of a rollout.** The configuration sets the working
folder, and the agent can see it. Each skill gets a different
`${HERMES_SKILL_DIR}` variable. A rollout runs in a sandbox, and the `task_id`
identifies the sandbox. The model and the reward function read the same
Workspace.

**[Continuity](../patterns/continuity.md): the stored history, the persona, the
model of the user and the links between Sessions.** Across Sessions, Hermes
keeps:

- the `SessionDB`, with full-text search across Sessions
- `SOUL.md` and `USER.md`
- the links between the stored Sessions across compressions.

In a Session, the memory prefetch adds related text from previous Sessions into
this turn. It does not write this text into the new history. The reviewer writes
the permanent part of Continuity. It examines the history of the last turns
after some turns and selects the data to write.

**[Topology](../patterns/topology.md): many objects in one process.** One Python
process contains many `AIAgent` objects at the same time. A profile, not a
process, isolates them. Hermes runs subagents in the same process with
`delegate_task`. Each frontend uses the same `AIAgent` class with the same
stored data in `HERMES_HOME`. Thus the frontends operate together.

The Frontend, the Session and the Environment can also be at different
locations. For example, the frontend is on a phone, the Session is on a small
server, and the Environment is where the terminal tool runs.

## Related

- [LangGraph case study](langgraph.md)
- [PI case study](pi.md)
