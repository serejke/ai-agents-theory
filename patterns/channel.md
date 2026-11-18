# Channel

A Channel is the method that agents in one run use to give their results to each
other. One agent writes data to an agreed location, for example a folder, and a
different agent reads the data from that location. Without a Channel, each agent
operates without the results of the other agents.

**Composition**: [Tool](../primitives/tool.md) set (`write`, `read`) + an agreed
format on a location that all the agents can read and write.

The location can be one of these:

- a filesystem path
- a queue
- an object that all the agents use
- a git branch
- the context window of the parent agent.

Channel is a pattern because of its function. It does not give a new function to
tools, because tools can write and read data. The pattern is that two agents
agree on the location and the names of the data. The
[variants](#channel-variants) are the different shapes of this function.

<a id="why-it-matters"></a>

## Value

The type of Channel controls how a pipeline of agents operates. In a pipeline,
one agent makes data, and a second agent must have this data. The method that
moves the data controls these properties of the pipeline:

- If the agents can run at the same time.
- If you can retry a phase that failed.
- If you can examine the data between the phases.
- If you can isolate one run from a different run.

[Memory](memory.md) and Channel keep data for different times and do different
work:

- Memory keeps knowledge across Sessions. An example is a fact that the agent
  found in a previous Session.
- Channel moves data in one run. An example is a fact that the research agent
  found and that the data agent must use next.

The two patterns use the same parts: a set of tools. Their function in the agent
system and the time that they keep the data are different.

<a id="formal-definition"></a>

## Definition

```typescript
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

## Channel variants

The variants of Channel are different in these properties:

- `direction`: the data moves in one direction or in the two directions.
- `schema`: the format of the data is set, agreed or free.
- `persistence`: the time that the data stays.
- `isolation`: the Channel keeps the data of one run away from a different run,
  or it does not.
- `acknowledgment`: the writer knows that the reader got the data, or the writer
  does not know it.

[The last section](#dimensions-for-choosing-a-channel) compares the variants in
one table.

<a id="1-filesystem"></a>

### Filesystem

The filesystem is the most easy Channel. Agents write files to agreed folders.
Other agents read these files.

```typescript
const filesystemChannel: Channel = {
  write: (key, value) => writeFile(`files/${phase}/${key}`, value),
  read: (pattern) => glob(pattern).map(readFile),
};
```

Example: in a research pipeline of agents, the research agents write their
results to `research_notes/`. A data agent reads these results and writes graphs
to `charts/`.

The data moves from the writer to the reader. The folder gives the phase, and
the file name tells the contents. The files stay after a crash of the process.

Strong points:

- It is very easy.
- A person can read the files.
- Each tool that reads or writes files can use it.

Weak points:

- No part of the system makes sure that the data agrees with a schema.
- The Channel does not isolate the runs. If no person cleans the folders, files
  from a previous run go into the next run.
- No part of the system finds errors. The reader reads all the data in the
  folder, also the bad data of a writer that failed.
- The writer does not know if the reader got the data.

<a id="2-return-value-agent-as-tool"></a>

### Subagent result (Agent as a Tool)

The subagent gives its result back to the parent agent as a tool result. The
parent agent then writes this result again in the prompt of the next subagent.

```typescript
const result = await researcherAgent.send("research quantum computing");
// The parent gets the result as a ToolResult and selects what to send next
await dataAnalyst.send(`Analyze this research: ${result}`);
```

Example: each system that uses an Agent as a Tool, for example the `Task` tool
of Claude Code.

In this variant, **the parent agent is the Channel**. It receives the output of
the first subagent and writes the input of the second subagent. If the system
also has a filesystem Channel, that Channel is only a second path. The primary
flow of data goes through the context of the parent agent.

The data moves from a subagent to the parent agent, and then to the next
subagent. The data has no schema: it is a string, and the LLM of the parent
agent gives it a meaning. The data stays only in the context window of the
parent agent. The Channel isolates each call from the other calls. If the run
continues, the parent agent got the result.

Strong points:

- The composition is easy.
- Before the parent agent sends the data, it can remove parts of the data, make
  the data shorter or write it again.
- The agents do not have data in one location that you must control.

Weak points:

- The context window of the parent agent sets the maximum quantity of data.
- The parent agent must continue to run for the full pipeline.
- When the parent agent makes the data shorter, it can remove important facts.

<a id="3-artifact"></a>

### Artifact

An artifact is an output with a type. One agent makes it, and a different agent
uses it. Different from a file, an artifact has metadata: a type, a schema and
the name of the agent that made it.

```typescript
type Artifact = {
  id: string;
  type: "research_note" | "chart" | "data_summary" | "report";
  content: string | Buffer;
  schema?: JSONSchema; // the shape of the content
  producedBy: string; // agent ID
  timestamp: Date;
};
```

Examples: the artifacts of Claude.ai, the Composer of Cursor, and the outputs of
the Replit agent.

The data moves in one direction. Each artifact has a type and can have a schema.
The artifacts usually stay for one Session, and the Session isolates them. The
writer does not know if the reader got the artifact.

Strong points:

- The reader knows the type of the data before it reads the data.
- A person can examine the artifacts.
- An artifact can hold metadata.

Weak points:

- It is necessary to have a registry or a store.
- It is more work than a filesystem Channel.

<a id="4-shared-state-object"></a>

### Shared state object

A shared state object is a data structure that two or more agents read and
change. Examples of this structure are a dictionary, a row of a database and a
document.

```typescript
type SharedState = {
  get: (key: string) => unknown;
  set: (key: string, value: unknown) => void;
  subscribe?: (key: string, callback: (value: unknown) => void) => void;
};
```

Examples: the shared state of LangGraph, the memory of CrewAI that all its
agents use, and systems in which all the agents use one blackboard.

The data moves in the two directions. The schema is different in each system:
the shared state of LangGraph has types, and a blackboard has no types. The data
stays for one run. If the system operates correctly, it isolates each run. A
reader can subscribe to a key and get each change of that key.

Strong points:

- It is flexible.
- The flow of data is not always linear. For example, agent 2 reads the data of
  agent 1 and changes it. Then agent 3 reads the changed data.
- It is a good fit for work that agents make better in many steps.

Weak points:

- Two agents can write the same key at the same time.
- It is not easy to debug, because you cannot see each change to the shared
  state.
- The shared state connects the agents. A change by one agent has an effect on
  the other agents.

Large data causes a problem in a system that keeps a copy of the full shared
state at each boundary between steps. [LangGraph](../case-studies/langgraph.md)
is an example. If the shared state holds a 50 MB artifact, each copy of the
shared state holds that 50 MB artifact again. The solution is to keep the data
in external storage and keep only a reference in the shared state. This solution
uses a filesystem Channel as a second path near the shared state. Thus one
system can use two or more variants of Channel.

<a id="5-message-queue--mailbox"></a>

### Message queue

A message queue lets agents send data at different times, without a connection
between them. Agents send messages to queues with names. Each agent reads its
messages when it can.

```typescript
type Mailbox = {
  send: (to: string, message: Message) => void;
  receive: (from?: string) => AsyncGenerator<Message>;
};
```

Examples: systems that use the actor model, and some frameworks of agents, for
example the conversations of AutoGen.

The data can move in the two directions and to many agents. Each message has a
type. A message stays until a reader reads it, or it stays permanently. The
system isolates each queue from the other queues. The reader can tell the writer
that it got the message or that it did not get the message.

Strong points:

- The agents are not connected. An agent can operate without knowledge of the
  other agents.
- Agents can run at different times or at the same time. The number of agents
  and their connections can change during the run.

Weak points:

- It is not easy to make and to operate.
- The sequence of the messages is important, and the queue must keep it.
- It is too much for a linear pipeline.

<a id="6-git"></a>

### Git

One agent commits changes. A different agent reads the diff or gets the result
from the repository.

```typescript
const gitChannel: Channel = {
  write: (key, value) => {
    writeFile(key, value);
    gitCommit(`Phase: ${phase}`);
  },
  read: (pattern) => gitDiff("HEAD~1", pattern),
};
```

Examples:

- GitHub Actions: one task commits, and a different task uses the commit.
- Coding agents that commit between the phase of the plan and the phase of the
  code.

The data moves in one direction. The data has no schema: it is files and an
agreed format for the commit messages. The data stays permanently, and git keeps
each version. Each branch keeps its data away from the other branches. The
writer does not know if the reader got the data.

Strong points:

- Git keeps each version, thus you can go back to the result of a previous
  phase.
- Branches isolate the runs.
- A person can examine each phase, for example in a pull request.

Weak points:

- Each git operation adds time and work.
- It is not a good fit for frequent messages.

<a id="dimensions-for-choosing-a-channel"></a>

## Select a Channel

Use these rules to select a Channel:

- If the pipeline is linear and has one run, use a filesystem Channel or the
  result of a subagent.
- If a person must examine the data between the phases, use a filesystem Channel
  or git.
- If two agents send the work back to each other many times, use a shared state
  object.
- If the number of agents changes and the agents run at different times, use a
  message queue.
- If you must go back to a previous phase, use git.
- If the data must agree with a schema, use artifacts, or a shared state object
  that examines the data.

This table compares the variants:

| Variant         | Direction       | Schema       | Data stays     | Isolated by  | Reader tells writer |
| --------------- | --------------- | ------------ | -------------- | ------------ | ------------------- |
| Filesystem      | One direction   | Agreed       | Permanently    | None         | No                  |
| Subagent result | One direction   | None         | In the context | Each call    | No                  |
| Artifact        | One direction   | Type         | One Session    | Each Session | No                  |
| Shared state    | Two directions  | Different    | One run        | Each run     | With subscribe      |
| Queue           | Many directions | Each message | Until read     | Each queue   | Yes                 |
| Git             | One direction   | None         | Each version   | Each branch  | No                  |

This type gives all the properties that help you select a Channel:

```typescript
type ChannelProperties = {
  direction: "uni" | "bi" | "broadcast";
  schema: JSONSchema | "convention" | "untyped";
  persistence: "ephemeral" | "session" | "run" | "permanent";
  isolation: "per-run" | "per-session" | "shared";
  acknowledgment: boolean;
  inspectable: boolean; // can a person see the data between phases?
  rollbackable: boolean; // can you remove the output of a phase?
};
```

## Related

- [Memory](memory.md): keeps knowledge across Sessions. A Channel moves data in
  one run.
- [Tool](../primitives/tool.md): the operations of an agent. A Channel is the
  connection between agents.
- [Workspace](workspace.md): composes Channel with Tool and PromptLoading to
  make a data area with a structure.
- [StateMachine](state-machine.md): gives the phases. Channels move the data
  between these phases.
