# Deployment

Deployment is the configuration that turns an abstract agent into a running system: what triggers it, what environment it runs in, where its output goes, and how multiple agents compose into teams. It explains why a CLI tool, an SDK, and a GitHub Action are the same agent with different wiring.

**Composition**: Trigger + [Session](session.md) + Output + Environment.

## Why It Matters

A [Session](session.md) is an agent that can have a conversation. But conversations don't start themselves. Sessions don't know where to write results. And the environment — whether the agent has access to a filesystem, a runtime, or the internet — isn't part of the Session abstraction. Deployment is the layer that answers: who starts the agent? What can it access? Where do results go?

Understanding Deployment as a separate concern reveals that Claude Code CLI, Claude Code SDK, and Claude Code GitHub Action are the same entity with different wiring — not different products.

## Formal Definition

```typescript
type Trigger =
  | { type: "cli_stdin"; input: string } // Claude Code CLI
  | { type: "sdk_call"; input: string } // Claude Code SDK
  | { type: "github_issue"; issue: GitHubIssue } // GitHub Action
  | { type: "github_comment"; comment: Comment } // GitHub Action
  | { type: "cron"; schedule: string } // scheduled
  | { type: "webhook"; payload: unknown }; // external signal

type OutputChannel =
  | { type: "cli_stdout" } // prints to terminal
  | { type: "github_pr"; repo: string } // creates PR
  | { type: "github_comment"; issueId: number } // writes comment
  | { type: "file"; path: string }; // writes to file

type AgentDeployment = {
  trigger: Trigger;
  session: Session; // or SessionFactory — new Session per trigger
  output: OutputChannel;
  environment: Environment;
};
```

---

## Environment — Three Layers

The agent's environment defines what resources it has access to. These layers don't have to be together — for many tasks only the first is enough:

```typescript
type Environment = {
  code: CodeAccess; // git clone / local files / read-only
  runtime?: Runtime; // npm, node, dev-server, build tools
  os?: OSAccess; // gh, git push, ssh, credentials
};

// "Light" agent — code only (planning, review, analysis)
const lightAgent: Environment = { code: gitClone("repo") };

// "Heavy" agent — full environment (build, test, deploy)
const heavyAgent: Environment = {
  code: gitClone("repo"),
  runtime: container({ image: "node:20", cmd: "npm run dev" }),
  os: shell({ credentials: ["gh_token", "ssh_key"] }),
};
```

The environment is orthogonal to the Session — the same Session configuration can run in a light environment (read-only code access for review) or a heavy environment (full build toolchain for implementation). This separation is what allows the same agent to be "a code reviewer" in one deployment and "a full-stack developer" in another.

---

## Same Agent, Different Deployments

Claude Code across deployment modes:

```typescript
// CLI — interactive, human in the loop
const cli: AgentDeployment = {
  trigger: { type: "cli_stdin" },
  session: claudeCodeSession,
  output: { type: "cli_stdout" },
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// SDK — programmatic, embedded in another system
const sdk: AgentDeployment = {
  trigger: { type: "sdk_call" },
  session: claudeCodeSession, // same session!
  output: { type: "file", path: "result.md" },
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// GitHub Action — event-driven, creates PRs
const action: AgentDeployment = {
  trigger: { type: "github_issue" },
  session: claudeCodeSession, // same session!
  output: { type: "github_pr", repo: "owner/repo" },
  environment: { code: gitClone("owner/repo"), runtime: container("node:20") },
};
```

The Session is identical across all three. The difference is entirely in the deployment configuration. This is a direct consequence of the compositional model — concerns are separated, so they can be recombined independently.

---

## Agent-as-Tool — The Composition Primitive

The result of one agent's work can become a tool result for another. This is the fundamental mechanism for agent composition:

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

This is exactly what Claude Code does with the Task tool — spawns a sub-agent (new Session) with a limited set of tools. The parent agent decides when to delegate, what to ask, and how to use the result. The sub-agent runs independently and returns its output through the [return value channel](../primitives/channel.md).

Agent-as-Tool composition is hierarchical: the parent controls the child, decides what to ask, interprets the result. The parent agent IS the [Channel](../primitives/channel.md) — it receives child A's output and formulates child B's input.

---

## Multi-Agent Topologies

When multiple agents work together, their communication structure determines the system's behavior, failure modes, and scalability. Four recurring topologies:

### Supervisor

One coordinator agent routes tasks to specialist agents and aggregates results:

```typescript
type Supervisor = {
  coordinator: Session; // routes, aggregates, decides
  specialists: Record<string, Session>; // domain experts
  // Communication: coordinator → specialist → coordinator (hub and spoke)
};
```

**Pros**: Clear authority. Coordinator maintains global context. Easy to add specialists.
**Cons**: Coordinator is a bottleneck and single point of failure. All context must pass through it.

### Swarm

Agents hand off directly to each other, peer-to-peer:

```typescript
type Swarm = {
  agents: Record<string, Session>;
  handoff: (from: string, to: string, context: unknown) => void;
  // Communication: any agent → any agent (mesh)
};
```

**Pros**: No bottleneck. Agents can specialize and transfer directly. Resilient — no single point of failure.
**Cons**: Hard to maintain global coherence. Debugging is difficult — no central place to observe the full flow.

See [Handoff](router.md) for the mechanism by which agents transfer control.

### Hierarchical

Nested supervisors — a supervisor delegates to sub-supervisors who manage their own specialist teams:

```typescript
type Hierarchical = {
  root: Supervisor;
  subTeams: Record<string, Supervisor>;
  // Communication: tree structure, each level manages its subtree
};
```

**Pros**: Scales to large agent counts. Each level manages bounded complexity.
**Cons**: Deep hierarchies add latency. Context gets lossy at each level — the root may not know details from leaf agents.

### GroupChat

All agents share a single conversation — each agent contributes based on its expertise:

```typescript
type GroupChat = {
  agents: Record<string, Session>;
  sharedHistory: Message[]; // all agents see the same conversation
  turnPolicy: (history: Message[], agents: string[]) => string; // who speaks next
};
```

**Pros**: All agents have full context. Natural for brainstorming, debate, multi-perspective analysis.
**Cons**: Context window pressure — every agent's contribution goes to every other agent. Doesn't scale beyond 3-5 agents.

### Choosing a Topology

| Factor       | Supervisor        | Swarm          | Hierarchical  | GroupChat      |
| ------------ | ----------------- | -------------- | ------------- | -------------- |
| Agent count  | 2-10              | 2-10           | 10+           | 2-5            |
| Coordination | Centralized       | Decentralized  | Tree          | Shared         |
| Context flow | Hub-and-spoke     | Point-to-point | Hierarchical  | Broadcast      |
| Failure mode | Coordinator fails | Incoherence    | Subtree fails | Token overflow |
| Best for     | Task delegation   | Expert handoff | Large teams   | Deliberation   |

---

## Related

- [Session](session.md) — what gets deployed
- [AgentLoop](agent-loop.md) — the computation inside each deployed agent
- [StateMachine](../primitives/state-machine.md) — phases and transitions within a deployment
- [Channel](../primitives/channel.md) — how agents pass data in multi-agent deployments
- [Router](router.md) — dispatch and handoff between agents
