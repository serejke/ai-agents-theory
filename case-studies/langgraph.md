# Case Study: LangGraph

LangGraph is a framework for agents. A developer writes an agent as a graph.
Each node of the graph is a step, and each edge goes to the next step.

Of the agent frameworks, LangGraph gives the most structure. This case study
does not tell that it is the best implementation. LangGraph is a good case study
because it tries to correct problems that small SDKs ignore. It shows that
important parts of the theory are correct. It also shows the problems of its
structure.

This document shows each concept of the theory in LangGraph. For each concept,
it gives the facts in two directions. LangGraph gives facts about agent systems,
and the theory gives facts about LangGraph.

<a id="what-langgraph-actually-is"></a>

## The parts of LangGraph

LangGraph puts these parts in one library. Each part is a concept of the theory:

- **A language for state machines**: nodes, edges and edges with conditions. It
  is the [StateMachine](../patterns/state-machine.md) pattern.
- **A shared state**: a `TypedDict` object that goes from node to node. Reducer
  functions merge the changes that two or more nodes make at the same time. It
  is the shared state variant of the
  [Channel](../patterns/channel.md#4-shared-state-object) pattern.
- **A small system that keeps the state**: it writes a checkpoint at each node
  boundary. A run can stop and then continue, also after a crash. This is the
  work of an orchestration system.
- **A tool loop**: `create_react_agent()` makes a graph with two nodes,
  `call_model` and `call_tools`. It is the [AgentLoop](../harness/agent-loop.md)
  harness.
- **A router of agents**: subgraphs and `Command(goto=X)` move a task from one
  agent to a different agent. It is the [Router](../patterns/router.md) pattern
  with [handoff](../patterns/router.md#handoff--routing-during-execution).

```
LangGraph = StateMachine + Channel(SharedState) + Checkpointing + AgentLoop + Router
```

LangGraph adds one API for all these parts. The sections below show where this
set of parts helps the developer. They also show where it is a limit.

<a id="langgraph-mapped-to-the-theory"></a>

<a id="statemachine--langgraphs-core"></a>

## StateMachine: the core of LangGraph

The `StateGraph` of LangGraph is the
[StateMachine](../patterns/state-machine.md) pattern in code. This table shows
the parts of the theory and the parts of LangGraph that agree:

```
Theory                          LangGraph
──────                          ─────────
Phase                           Node
Transition                      Edge
Conditional transition          Conditional edge
Guard                           Edge condition function
human_approval trigger          interrupt() + Command(resume=)
PhaseConfig per phase           Different logic per node
```

LangGraph makes a full framework around StateMachine. This shows that
StateMachine is an important pattern. StateMachine composes these parts:

- a [Guardrail](../harness/guardrail.md) that lets each phase use only its tools
- a phase variable
- the rules that change the phase.

LangGraph gives these parts as one graph language.

<a id="2-the-graph-is-one-implementation-of-statemachine"></a>

### The graph is one implementation

The graph is the value that LangGraph gives to developers. The concept of the
graph is the StateMachine pattern: phases with different functions, and rules
that change the phase. A graph is one method to write a StateMachine. Usual code
that keeps the phase in a variable is a different method.
[StateMachine](../patterns/state-machine.md#implementation-approaches-graph-vs-imperative)
compares the two methods.

<a id="6-the-anti-pattern-warning-applies"></a>

### Edge conditions that the LLM selects

In a production system, make each edge condition examine the state. Do not make
the LLM select the next node.

The
[StateMachine](../patterns/state-machine.md#anti-pattern-prompt-enforced-orchestration)
document tells about a frequent error, "Prompt-Enforced Orchestration". In this
error, the prompt gives the sequence of the phases, not the code. The graph of
LangGraph prevents the clear version of this error, because its nodes give the
sequence in code. But a different version of the error can occur: an edge
condition that an LLM call selects, with no check of the state.

```python
# Looks like code-level routing, but is actually prompt-enforced:
graph.add_conditional_edges("verify", lambda s: llm("should we proceed?"))
```

This edge is in the code, but a prompt selects the next step. Thus the result of
the condition is not deterministic. An edge condition that examines the data of
the state is deterministic.

<a id="agentloop--modeled-as-a-graph-abstraction-mismatch"></a>

<a id="1-the-tool-use-loop-is-a-while-loop"></a>

## AgentLoop: a loop as a graph

For the basic tool loop, a `while` loop is a better fit than a graph.

`create_react_agent()` makes a graph with two nodes, `call_model` and
`call_tools`. The graph goes from one node to the other node until the model
gives text without a tool call. This is the
[AgentLoop](../harness/agent-loop.md) harness: an LLM call, then the tool calls,
then the next LLM call.

The theory shows the AgentLoop as a `while` loop. Anthropic also writes it as a
`while` loop of approximately 30 lines:

```
while true:
    prompt = assemble(system, messages, tool_schemas)
    text, tool_calls = parse(llm(prompt))
    if not tool_calls: return text
    results = execute(tool_calls)
    messages.append(results)
```

A graph adds a step that compiles it, node definitions and edge routing. The
work stays the same: call the LLM, and if it gives tool calls, run them and do
it again. The graph helps in a workflow with many phases, where the shape of the
graph is important. For the basic tool loop, use a `while` loop.

<a id="channel--restricted-to-sharedstate"></a>

<a id="2-channel-variants-beyond-shared-state"></a>

## Channel: only shared state

LangGraph gives only one [Channel](../patterns/channel.md) variant: the
[shared state](../patterns/channel.md#4-shared-state-object). A `TypedDict`
object goes from node to node. Each node receives the full state and gives back
a change to some of its fields.

The theory gives more
[Channel variants](../patterns/channel.md#channel-variants):

- filesystem
- return value
- artifact
- shared state
- mailbox
- versions in git.

Each variant keeps data for a different time. Each variant also isolates the
agents with a different method. And each variant tells with a different method
that the data got to the other agent. Agent systems use different channels for
different shapes of data.

LangGraph users make the other channels in their code, not with the framework.
Examples are artifacts in files, messages in a mailbox, and handoff with
versions in a git repository.

### Large artifacts

Large artifacts show the limit of one channel most clearly. LangGraph writes the
full state in a checkpoint at each node boundary. Each checkpoint is a new
snapshot that does not change. If a node puts a 50 MB PDF in the state,
LangGraph serializes these 50 MB into each subsequent checkpoint.

With 15 nodes and 100 runs at the same time, this quantity of data is too large.
The large checkpoints cause these problems in Postgres:

- The `WAL` becomes much larger than the data.
- The `TOAST` compression starts again and again.
- The copies of the database do not get the changes in time.

LangGraph recommends a different method: keep a reference in the state, and keep
the data in external storage. For example, the first node sends the PDF to S3
and writes `{"pdf_url": "s3://bucket/report.pdf"}` to the state. The second node
reads the URL and fetches the artifact.

This method operates. But the node then uses a filesystem channel or an object
store channel together with the shared state channel of LangGraph. The framework
has only one channel, thus users make a second channel in their code. Thus most
LangGraph systems that are not small keep large artifacts in external storage,
for example S3, Redis or a local disk. The `TypedDict` state holds only the
references.

Two other features do not prevent this problem:

- **The `Store` of LangGraph** keeps Memory across Sessions. It is a store of
  JSON documents that can find documents by their meaning. It is for small
  memories with a structure, for example the selections of a user. It is not for
  large files.
- **LangChain tools with `response_format="content_and_artifact"`** keep large
  data out of the context window of the LLM. But the artifact stays in the
  message history, and LangGraph writes it in each checkpoint. This prevents the
  problem of tokens, but not the problem of storage.

<a id="checkpointing--bundled-not-separated"></a>

## Checkpoints: orchestration in the library

LangGraph writes the state in a checkpoint at each node boundary. These
checkpoints give three functions:

- A run continues after a crash.
- A person can accept a step before it occurs. The run stops, keeps its state,
  waits and then continues.
- A developer can debug from a previous checkpoint ("time-travel debugging").

The theory puts checkpoints in the orchestration system, apart from the code of
the agent. LangGraph puts them in its run model. The developer gets checkpoints
and does not deploy a different orchestration system. But the agent code then
also holds the orchestration. This is a decision of LangGraph.

<a id="1-concern-bundling-is-a-legitimate-architecture"></a>

### When one library is sufficient

For some systems, one library for orchestration, state and steps is a correct
structure. These systems do much reasoning, but their steps are easy. Examples
are chat assistants, research agents and coding agents. If no dangerous
operation must continue after a crash, a different orchestration system only
adds work.

The theory gives orchestration and the agent code as two different parts. A
system can keep the two parts in one library or in two systems.
[StateMachine](../patterns/state-machine.md#concern-bundling-vs-separation)
tells about this decision.

<a id="4-orchestration-and-execution-will-eventually-separate"></a>

### When orchestration becomes a different system

When a system becomes larger, its orchestration and its steps go into different
systems. One library is sufficient until the system must have one of these
functions:

- runs at scheduled times, for example with cron
- runs that a different system starts
- tasks across two or more services
- operations of the company that must continue after a crash.

Then LangGraph becomes one component of a larger system. It controls the layer
of the reasoning. The theory shows this
[separation](../patterns/state-machine.md#the-cognitive--execution-layer-split)
into layers. A system that starts with one framework divides into parts when it
becomes larger.

<a id="memory-vs-state--a-clean-separation"></a>

## Memory and state

LangGraph keeps two types of data apart, and the theory does the same:

- **The `checkpointer`** keeps the state of one run, for example the
  conversation history and the results between the steps. It is the
  [Session](../harness/session.md) history and a
  [Channel](../patterns/channel.md).
- **The `Store`** (`BaseStore`, or `PostgresStore` with `pgvector`) keeps
  knowledge across threads and across Sessions. It is the
  [Memory](../patterns/memory.md) pattern.

In the theory, a Channel moves data in one run, and Memory keeps knowledge
across Sessions. LangGraph keeps these two parts apart correctly.

<a id="3-memory-is-more-than-a-store"></a>

### Memory is more than a store

The `Store` gives a location for the data. It does not select the data to keep,
the time to write it, or the method to find it again. These decisions are the
hard part of Memory. The [Memory](../patterns/memory.md#types-of-memory) pattern
gives them:

- the types of memory: memory of previous Sessions, memory of facts and memory
  of procedures
- the methods to write data and the methods to find data
- the levels of memory: for all work, for one project and for one task.

<a id="3-handoff-is-a-distinct-routing-pattern"></a>

## Router: handoff between agents

In LangGraph, `Command(goto="other_agent", graph=Command.PARENT)` moves the
control from one agent to an equal agent. This handoff is not a call to a
subagent, where an agent gives a task to an agent below it. It is also not the
selection of an agent before the run starts.
[Router](../patterns/router.md#handoff--routing-during-execution) gives handoff
as a type of routing.

<a id="4-multi-agent-topologies-recur"></a>

LangGraph also gives these structures for groups of agents:

- [one agent that controls a group](../patterns/topology.md#supervisor-hub)
- [a group of equal agents](../patterns/topology.md#swarm-mesh)
- [a tree of groups](../patterns/topology.md#hierarchical-tree).

Many agent systems use these structures. The [Topology](../patterns/topology.md)
pattern gives them.

<a id="5-every-concept-boundary-is-a-verification-surface"></a>

## Verification

Each boundary between two concepts is a point where a test can examine the
system. [Verification](../verification/index.md) calls these points
[seams](../verification/seams.md). In LangGraph, the node boundaries, the edge
conditions and the state changes are seams.

LangSmith records traces and examines them. The
[verification stack](../verification/index.md#the-verification-stack) of the
theory adds more layers:

- tests with an LLM that gives known outputs
- snapshot tests of the prompts
- checks of the properties of the output
- checks of the sequence of the steps of the agent.

All these layers together give a full method to examine an agent.

<a id="summary"></a>

## The case study in short

LangGraph is the [StateMachine](../patterns/state-machine.md) pattern as a
product. It also contains a [Channel](../patterns/channel.md) with shared state,
checkpoints and the [AgentLoop](../harness/agent-loop.md) harness. It shows that
StateMachine is an important pattern.

<a id="what-langgraph-reveals"></a>

LangGraph shows these structures in operation:

- handoff between agents
- structures for groups of agents
- one library for orchestration, state and steps.

The theory gives each structure a position. [Topology](../patterns/topology.md)
is a pattern. [Router](../patterns/router.md) gives handoff. StateMachine gives
the decision about one library.

<a id="what-the-theory-reveals-about-langgraph"></a>

The theory divides into parts the functions that LangGraph puts together. The
two are not in conflict, because they operate at different levels. The theory
gives the words for the parts of LangGraph. It also shows when one library helps
the developer and when it is a limit.

## Related

- [StateMachine](../patterns/state-machine.md): the core of LangGraph, and a
  graph compared with usual code.
- [Channel](../patterns/channel.md): all the variants, and the shared state
  variant of LangGraph.
- [AgentLoop](../harness/agent-loop.md): a `while` loop and a graph.
- [Memory](../patterns/memory.md): all the parts of Memory, and the `Store` of
  LangGraph.
- [Verification](../verification/index.md): the full verification stack, with
  more than traces.
