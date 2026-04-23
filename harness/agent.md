# Agent

An Agent is the runnable unit: a Session wrapped in a Frontend and pinned to an Environment. It is what people mean when they say "I built an agent" or "the Claude Code agent" or "a Slack agent" — the whole invocable thing, not just the reasoning inside it. "Claude Code CLI," "Claude Code SDK," and "Claude Code GitHub Action" are three Agents built from the same Session with different Frontends and Environments.

**Composition**: Frontend + [Session](session.md) + [Environment](../environment/environment.md).

## Why It Matters

A [Session](session.md) can hold a conversation, but it doesn't start itself, doesn't know where to send results, and doesn't know what substrate it's running on. The Agent is the layer that answers: what wraps this Session? What's the I/O contract with the caller? Which Environment does it run against?

Agent is harness, not a compositional choice. Every running agent system is an Agent, even if it's just `python agent.py < input.txt > output.txt` — that is a Frontend (stdin/stdout) wrapping a Session in the local Environment. Thinking in terms of Agent makes it obvious that moving a coding assistant from "interactive CLI" to "scheduled GitHub Action" is not a rewrite; it's a swap of Frontend and Environment around the same Session.

### Agent vs. agent

The theory uses **Agent** (capital A) for this formal concept — the runnable unit, the triple (Frontend, Session, Environment). Lowercase "agent" is reserved for colloquial use: "the agent decides which tool to call" refers to the reasoning, which happens inside the Session. This mirrors the Tool-vs-tool convention elsewhere in the book.

When precision matters, the theory names the specific layer: LLM for the reasoner, Session for the cognitive core (AgentLoop + history + tools + system prompt), Agent for the whole invocable unit.

## Formal Definition

```typescript
type Frontend =
  | { kind: "cli"; interactive: boolean } // stdin/stdout, with or without prompt loop
  | { kind: "sdk"; module: string } // function call from another program
  | { kind: "http"; route: string; verb: string } // REST/GraphQL endpoint
  | { kind: "github_action"; events: string[] } // issue_opened, pr_commented, etc.
  | { kind: "slack_bot"; channels: string[] } // incoming message
  | { kind: "cron"; schedule: string } // scheduled fire
  | { kind: "webhook"; url: string }; // arbitrary external signal

type Agent = {
  frontend: Frontend;
  session: Session; // or a factory: new Session per invocation
  environment: Environment;
};
```

The Frontend carries both the invocation side (what event or call triggers the run) and the output side (how results reach the caller) — these are two faces of the same interface. A CLI Frontend reads stdin and writes stdout. An HTTP Frontend receives a request and sends a response. A GitHub Action Frontend consumes an event payload and writes a PR or comment. There's no need to separate "trigger" and "output" as independent concepts; they are paired halves of the Frontend contract.

---

## Same Session, Different Agents

A fixed Session configuration can be wrapped into multiple Agents with no change to the cognitive core:

```typescript
const claudeCodeSession = createSession({
  llm: claude_sonnet,
  tools: [fileRead, fileWrite, bash, grep, glob, webFetch],
  system: claudeCodeSystemPrompt,
});

// Interactive CLI — human in the loop
const cli: Agent = {
  frontend: { kind: "cli", interactive: true },
  session: claudeCodeSession,
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// SDK — embedded in another program
const sdk: Agent = {
  frontend: { kind: "sdk", module: "claude-code-sdk" },
  session: claudeCodeSession, // same Session
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// GitHub Action — event-driven, creates PRs
const action: Agent = {
  frontend: { kind: "github_action", events: ["issues.opened"] },
  session: claudeCodeSession, // same Session
  environment: { code: gitClone("owner/repo"), runtime: container("node:20") },
};
```

The Session is identical across all three. What changes is the Frontend (how the outside world invokes and receives results) and the Environment (the substrate the agent acts on). This is a direct consequence of keeping concerns separate: the cognitive core doesn't know how it was invoked or where its output goes, so the same core serves every Agent.

---

## The Frontend Is the Product Surface

What a user experiences as "Claude Code" or "Codex" or "Cursor" is the Frontend. The cognitive core is invisible to the user; the Frontend is what they interact with. Product differentiation — the UX, the I/O shape, the integration points — lives in the Frontend layer while the Session remains commodity.

This is why the same underlying Session can appear as:

- An interactive terminal Agent
- A VS Code extension Agent embedded in an editor
- A web Agent that talks to a backend which talks to the Session
- A GitHub bot Agent that opens PRs from issues
- A Slack bot Agent that answers questions about a codebase
- A scheduled Agent that runs overnight and emails a report

Each is a different product — a different Agent. The Session inside is the same underlying capability.

---

## Agent vs. Environment

An Agent _uses_ an [Environment](../environment/environment.md) but doesn't define it. Two Agents with the same Frontend and Session but different Environments are meaningfully different: a "coding Agent with full local access" is different from "the same coding Agent in a sandboxed container," even though the Frontend and cognitive core are identical.

Keep the layers distinct:

- **Frontend** = how the outside world invokes and receives from the Agent.
- **Session** = the cognitive core (AgentLoop + history + tools + system prompt).
- **Environment** = the substrate the Agent acts on (filesystem, runtime, network, credentials).

An Agent pins all three down into a concrete runnable unit.

---

## Agent vs. Topology

A single Agent wraps one Session. When multiple Agents coordinate, you have a [Topology](../patterns/topology.md) — a pattern layered on top of multiple Agents. Each participant in a Supervisor or Swarm is its own Agent (its own Frontend wiring to its own Session and Environment); the Topology describes how those Agents communicate.

Agent is the unit. Topology is the arrangement of units.

---

## Related

- [Session](session.md) — the cognitive core that the Agent wraps
- [AgentLoop](agent-loop.md) — the computation inside the Session
- [Environment](../environment/environment.md) — the substrate the Agent pins the Session to
- [Topology](../patterns/topology.md) — the pattern that coordinates multiple Agents
- [StateMachine](../patterns/state-machine.md) — phases and transitions within the Session
