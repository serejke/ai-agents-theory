# Topology

A Topology is the coordination shape of a multi-agent system — how multiple agents communicate, who speaks to whom, where control transfers, where context flows. It is the pattern that answers "how are N agents arranged?" once you have more than one.

**Composition**: Multiple [Session](../harness/session.md) instances + [Channel](channel.md) (data flow between them) + [Router](router.md) (control flow between them), arranged in a coordination shape (hub, mesh, tree, broadcast).

## Why It Matters

A single-agent system has no Topology — there is no one to coordinate with. As soon as a system has multiple agents, the shape of their communication determines system behavior: whether they can run in parallel, where bottlenecks appear, how context is preserved, which failure modes exist.

Topology is a compositional choice, not a harness concept. Some systems have it (supervisor-with-specialists, peer-swarm, hierarchical teams); many don't (single-agent coding tools, single-agent chatbots). When a system does have it, the choice of shape — centralized vs. decentralized, flat vs. nested, point-to-point vs. broadcast — is the most consequential architectural decision in the design.

## Formal Definition

```typescript
type Topology = {
  agents: Record<string, Session>; // the agents themselves
  channels: Channel[]; // how data flows between them
  routing: Router; // how control transfers between them
  shape: "hub" | "mesh" | "tree" | "broadcast"; // the coordination pattern
};
```

The `shape` field is what names the Topology variant. The ingredients (agents, channels, routing) are common to all variants; the shape determines how they're wired.

---

## Variants

### Supervisor (Hub)

One coordinator agent routes tasks to specialist agents and aggregates results. Communication is hub-and-spoke.

```typescript
type Supervisor = {
  coordinator: Session; // routes, aggregates, decides
  specialists: Record<string, Session>; // domain experts
  // Communication: coordinator → specialist → coordinator
};
```

**Pros**: Clear authority. Coordinator maintains global context. Easy to add specialists.
**Cons**: Coordinator is a bottleneck and single point of failure. All context must pass through it.

The specialist-invocation mechanism is typically [Agent-as-Tool](#agent-as-tool): the coordinator calls a specialist as if it were a Tool, receives the result, and incorporates it into its own reasoning.

### Swarm (Mesh)

Agents hand off directly to each other, peer-to-peer. No central coordinator.

```typescript
type Swarm = {
  agents: Record<string, Session>;
  handoff: (from: string, to: string, context: unknown) => void;
  // Communication: any agent → any agent
};
```

**Pros**: No bottleneck. Agents can specialize and transfer directly. Resilient — no single point of failure.
**Cons**: Hard to maintain global coherence. Debugging is difficult — no central place to observe the full flow.

Handoff mechanics are described in the [Router](router.md) pattern — the same dispatch machinery used within a single agent, generalized across agents.

### Hierarchical (Tree)

Nested supervisors — a supervisor delegates to sub-supervisors who manage their own specialist teams.

```typescript
type Hierarchical = {
  root: Supervisor;
  subTeams: Record<string, Supervisor>;
  // Communication: tree structure, each level manages its subtree
};
```

**Pros**: Scales to large agent counts. Each level manages bounded complexity.
**Cons**: Deep hierarchies add latency. Context gets lossy at each level — the root may not know details from leaf agents.

### GroupChat (Broadcast)

All agents share a single conversation — each agent contributes based on its expertise.

```typescript
type GroupChat = {
  agents: Record<string, Session>;
  sharedHistory: Message[]; // all agents see the same conversation
  turnPolicy: (history: Message[], agents: string[]) => string; // who speaks next
};
```

**Pros**: All agents have full context. Natural for brainstorming, debate, multi-perspective analysis.
**Cons**: Context window pressure — every agent's contribution goes to every other agent. Doesn't scale to many participants.

### Choosing a Topology

| Factor       | Supervisor        | Swarm          | Hierarchical  | GroupChat      |
| ------------ | ----------------- | -------------- | ------------- | -------------- |
| Coordination | Centralized       | Decentralized  | Tree          | Shared         |
| Context flow | Hub-and-spoke     | Point-to-point | Hierarchical  | Broadcast      |
| Failure mode | Coordinator fails | Incoherence    | Subtree fails | Token overflow |
| Best for     | Task delegation   | Expert handoff | Large teams   | Deliberation   |

---

## Agent-as-Tool

The fundamental composition mechanism that underpins most Topologies: a Session can be wrapped as a [Tool](../primitives/tool.md) and invoked by another Session.

```typescript
const subAgent: Tool = {
  name: "research_codebase",
  execute: async (params) => {
    const sub = createSession({ llm, tools: [fileRead, grep, glob] });
    const result = await sub.send(params.question);
    return result; // returned as ToolResult in parent session
  },
};
```

Claude Code's Task tool is exactly this: spawn a sub-agent with a limited tool set, receive its output as a tool result. Supervisor and Hierarchical topologies are built on this mechanism — the coordinator invokes specialists as Agent-as-Tool.

When Agent-as-Tool is the composition used, the parent agent _is_ the [Channel](channel.md) — it receives child A's output and formulates child B's input. The parent's context window carries inter-agent state.

---

## Related

- [Session](../harness/session.md) — each agent in a Topology is a Session
- [Channel](channel.md) — the data-flow ingredient of Topology
- [Router](router.md) — the control-flow ingredient of Topology, including Handoff
- [Agent](../harness/agent.md) — each participant in a Topology is its own Agent
- [LangGraph case study](../case-studies/langgraph.md) — supervisor/swarm/hierarchical topologies as a framework feature
