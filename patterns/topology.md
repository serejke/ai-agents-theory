# Topology

A Topology is the shape of a system with two or more agents. It tells which
agent speaks to which agent, where the control goes from one agent to a
different agent, and where the context goes. For example, one agent can give
tasks to other agents and collect their results.

**Composition**: Two or more [Sessions](../harness/session.md) +
[Channel](channel.md) (the flow of data between them) + [Router](router.md) (the
flow of control between them). A shape connects them: hub, mesh, tree or
broadcast.

<a id="why-it-matters"></a>

## Value

If a system has two or more agents, the shape of their connections controls how
the system operates. The shape controls these properties:

- which agents can run in parallel
- where a bottleneck occurs
- how the system keeps the context
- which errors can occur.

A system with one agent has no Topology, because no other agent connects to it.

Topology is a pattern, not a part of the harness. Some systems use it, for
example a Supervisor with specialists, a Swarm of agents or a hierarchy of
teams. Many systems do not use it, for example coding agents and chat assistants
with one agent each.

If a system uses a Topology, its shape is the most important decision about the
structure of the system. The shape has these decisions:

- one agent that controls all the agents, or no such agent
- one level, or levels in levels
- messages from one agent to one agent, or a broadcast to all agents.

<a id="formal-definition"></a>

## Definition

```typescript
type Topology = {
  agents: Record<string, Session>; // the agents
  channels: Channel[]; // how data flows between them
  routing: Router; // how control goes between them
  shape: "hub" | "mesh" | "tree" | "broadcast"; // the coordination pattern
};
```

The `shape` field gives the name of the Topology variant. All variants have the
same parts: agents, Channels and routing. The shape tells how these parts
connect.

<a id="variants"></a>

## Variants

<a id="supervisor-hub"></a>

### Supervisor: a hub

One coordinator agent sends tasks to specialists and collects their results. All
messages go through the coordinator, which is the hub.

```typescript
type Supervisor = {
  coordinator: Session; // sends tasks, collects results, makes decisions
  specialists: Record<string, Session>; // experts of one area each
  // Messages: coordinator → specialist → coordinator
};
```

Strong points:

- The authority is clear.
- The coordinator keeps the context of all the system.
- It is easy to add a specialist.

Weak points:

- The coordinator is a bottleneck. If it stops, all the system stops.
- All the context must go through the coordinator.

The coordinator usually calls a specialist with the
[Agent-as-Tool](#agent-as-tool) mechanism. It calls the specialist as a Tool,
receives the result and uses it in its reasoning.

<a id="swarm-mesh"></a>

### Swarm: a mesh

The agents give tasks directly to each other. No coordinator controls them.

```typescript
type Swarm = {
  agents: Record<string, Session>;
  handoff: (from: string, to: string, context: unknown) => void;
  // Messages: any agent → any agent
};
```

Strong points:

- There is no bottleneck.
- Each agent can do one type of task and give the task directly to a different
  agent.
- If one agent stops, the other agents continue.

Weak points:

- It is hard to make sure that all the agents agree about the full task.
- It is hard to debug, because no one point shows the full flow.

The [Router](router.md) pattern tells how a handoff operates. A Swarm uses the
same routing mechanism as one agent, but between agents.

<a id="hierarchical-tree"></a>

### Hierarchy: a tree

A Supervisor gives tasks to lower Supervisors. Each lower Supervisor controls
its team of specialists.

```typescript
type Hierarchical = {
  root: Supervisor;
  subTeams: Record<string, Supervisor>;
  // Messages: a tree. Each level controls the levels below it.
};
```

Strong points:

- It can operate with many agents.
- Each level controls only a small part of the work.

Weak points:

- Many levels increase the latency.
- Each level removes some of the context. The root agent possibly does not know
  the facts that the lowest agents found.

<a id="groupchat-broadcast"></a>

### GroupChat: a broadcast

All the agents use the same conversation. Each agent adds its knowledge to the
conversation.

```typescript
type GroupChat = {
  agents: Record<string, Session>;
  sharedHistory: Message[]; // all agents see the same conversation
  turnPolicy: (history: Message[], agents: string[]) => string; // who speaks next
};
```

Strong points:

- All the agents have the full context.
- It is a good shape to find new solutions, to compare different solutions and
  to examine a problem from different views.

Weak points:

- The context window fills quickly, because each message of an agent goes to all
  the other agents.
- It is not satisfactory for many agents.

<a id="choosing-a-topology"></a>

### How to select a Topology

| Property     | Supervisor            | Swarm               | Hierarchy    | GroupChat        |
| ------------ | --------------------- | ------------------- | ------------ | ---------------- |
| Control      | One coordinator       | No coordinator      | Tree         | One conversation |
| Context flow | Hub                   | Agent to agent      | Tree         | Broadcast        |
| Errors       | Coordinator stops     | Agents do not agree | A team stops | Too many tokens  |
| Best for     | Tasks for specialists | Handoffs            | Large teams  | Different views  |

## Agent-as-Tool

Agent-as-Tool is the basic mechanism of most Topologies. A Session operates as a
[Tool](../primitives/tool.md), and a different Session calls it.

```typescript
const subAgent: Tool = {
  name: "research_codebase",
  execute: async (params) => {
    const sub = createSession({ llm, tools: [fileRead, grep, glob] });
    const result = await sub.send(params.question);
    return result; // the parent Session receives it as a tool result
  },
};
```

The Task tool of Claude Code is this mechanism. It starts a subagent with a
small set of tools and gives the output of the subagent back as a tool result.
The Supervisor and the hierarchy use this mechanism: the coordinator calls each
specialist as a Tool.

With Agent-as-Tool, the agent that calls the subagents is the
[Channel](channel.md). It receives the output of one subagent and writes the
input of the next subagent. The context window of that agent keeps the data that
goes between the agents.

## Related

- [Session](../harness/session.md): each agent in a Topology is a Session.
- [Channel](channel.md): the part of a Topology that moves the data.
- [Router](router.md): the part of a Topology that moves the control. It also
  gives the handoff.
- [Agent](../harness/agent.md): each agent in a Topology is an Agent.
- [LangGraph case study](../case-studies/langgraph.md): a framework that gives
  the Supervisor, the Swarm and the hierarchy as features.
