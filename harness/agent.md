# Agent

An Agent is a full program that a person or a different program can start and
use, for example Claude Code in a terminal. It has a frontend for the input and
the output, a loop with an LLM and tools, and resources for the tools. The
theory calls the loop the **Session** and the resources the **Environment**.

**Composition**: Frontend + [Session](session.md) +
[Environment](../environment/environment.md).

<a id="why-it-matters"></a>

## Value

A [Session](session.md) can hold a conversation, but a different part must start
it. The Session does not know where to send its results. It also does not know
the resources that it operates on. The Agent gives these facts:

- the frontend that starts the Session
- the contract of the input and the output with the caller
- the [Environment](../environment/environment.md) of the Session.

The Agent is a part of the harness, not an optional pattern. Each agent system
that runs is an Agent. A small script, for example
`python agent.py < input.txt > output.txt`, is also an Agent. Its frontend is
the standard input and the standard output, and its Session runs in the local
Environment.

With this concept, a change of frontend is easy to see. If you move a coding
agent from a CLI to GitHub Actions, you do not write it again. You replace the
frontend and the Environment, and the Session stays the same. For example,
"Claude Code CLI", "Claude Code SDK" and "Claude Code GitHub Action" are
different Agents. They use the same Session with different frontends and
different Environments.

<a id="formal-definition"></a>

## Definition

```typescript
type Frontend =
  | { kind: "cli"; interactive: boolean } // stdin/stdout, with or without a prompt loop
  | { kind: "sdk"; module: string } // function call from another program
  | { kind: "http"; route: string; verb: string } // REST or GraphQL endpoint
  | { kind: "github_action"; events: string[] } // issue_opened, pr_commented, ...
  | { kind: "slack_bot"; channels: string[] } // incoming message
  | { kind: "cron"; schedule: string } // runs on a schedule
  | { kind: "webhook"; url: string }; // any external signal

type Agent = {
  frontend: Frontend;
  session: Session; // or a factory: a new Session for each run
  environment: Environment;
};
```

The frontend has two sides of one interface:

- The input side: the call or the message that starts the run.
- The output side: the path of the results back to the caller.

| Frontend       | Input side         | Output side                 |
| -------------- | ------------------ | --------------------------- |
| CLI            | the standard input | `stdout`                    |
| HTTP           | a request          | a response                  |
| GitHub Actions | a payload          | a pull request or a comment |

The theory does not divide the start of the run and the output into two
concepts. They are the two sides of the contract of the frontend.

<a id="agent-vs-environment"></a>

## The parts of an Agent

An Agent connects these layers into one unit that runs:

- **Frontend**: how a person or a program starts the Agent and gets its results.
- **[Session](session.md)**: the reasoning core. It holds the
  [AgentLoop](agent-loop.md), the history, the tools and the system prompt.
- **Environment**: the resources that the Agent operates on. These are the
  filesystem, the runtime, the network and the credentials.

An Agent uses an [Environment](../environment/environment.md), but the
Environment is not a part of the Agent. If two Agents have the same frontend and
the same Session but different Environments, they are different Agents. For
example, a coding Agent with full access to the local computer is different from
the same coding Agent in a sandbox container.

<a id="agent-vs-agent"></a>

## Agent and agent

When a person tells "I built an agent", "the Claude Code agent" or "a Slack
agent", the person refers to the Agent. This is the full system that the person
can start, not only the reasoning in it.

The theory writes **Agent** with an uppercase letter for this concept: the unit
of a frontend, a Session and an Environment. It writes "agent" in lowercase for
the usual meaning. In "the agent decides which tool to call", the word refers to
the reasoning, and this reasoning occurs in the Session. The book uses the same
rule for Tool and tool.

If the layer is important, the theory gives its name:

- **LLM**: the part that does the reasoning.
- **Session**: the reasoning core. It holds the AgentLoop, the history, the
  tools and the system prompt.
- **Agent**: the full unit that a person or a program can start.

<a id="same-session-different-agents"></a>

## One Session in different Agents

You can put one Session configuration into many Agents, and the reasoning core
does not change:

```typescript
const claudeCodeSession = createSession({
  llm: claude_sonnet,
  tools: [fileRead, fileWrite, bash, grep, glob, webFetch],
  system: claudeCodeSystemPrompt,
});

// Interactive CLI: a human in the loop
const cli: Agent = {
  frontend: { kind: "cli", interactive: true },
  session: claudeCodeSession,
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// SDK: embedded in another program
const sdk: Agent = {
  frontend: { kind: "sdk", module: "claude-code-sdk" },
  session: claudeCodeSession, // same Session
  environment: { code: localFS, runtime: localNode, os: localShell },
};

// GitHub Action: starts on repository events, creates PRs
const action: Agent = {
  frontend: { kind: "github_action", events: ["issues.opened"] },
  session: claudeCodeSession, // same Session
  environment: { code: gitClone("owner/repo"), runtime: container("node:20") },
};
```

The Session is the same in the three Agents. Only these parts change:

- the frontend: how a person or a program starts the Agent and gets the results
- the Environment: the resources that the agent operates on.

This result comes from the separation of the layers. The reasoning core does not
know how the run started or where its output goes. Thus the same core can
operate in each Agent.

<a id="the-frontend-is-the-product-surface"></a>

## The frontend is the product

A user knows Claude Code, Codex or Cursor through its frontend. The user does
not see the reasoning core. The products are different in their frontends: the
user interface, the format of the input and the output, and the connections to
other systems. The Session can be the same in many products.

Thus the same Session can be in these Agents:

- an Agent in a terminal
- an Agent in a VS Code extension, where the user changes code
- a web Agent that sends requests to a backend, which then sends them to the
  Session
- an Agent on GitHub that opens pull requests from issues
- an Agent in Slack that replies to messages about the code of a project
- an Agent that runs after the work hours and sends a report in an email.

Each item is a different product and a different Agent. The Session in them
gives the same functions.

<a id="agent-vs-topology"></a>

## Agent and Topology

One Agent holds one Session. When two or more Agents operate together, they make
a [Topology](../patterns/topology.md). A Topology is a pattern on top of these
Agents. Each Session in a Supervisor or a Swarm is in a different Agent, with a
frontend, a Session and an Environment. The Topology tells how these Agents send
data to each other.

The Agent is the unit. The Topology is the configuration of the units.

## Related

- [Session](session.md): the reasoning core in the Agent
- [AgentLoop](agent-loop.md): the loop in the Session
- [Environment](../environment/environment.md): the resources that the Agent
  connects the Session to
- [Topology](../patterns/topology.md): the pattern that connects two or more
  Agents
- [StateMachine](../patterns/state-machine.md): the phases in the Session and
  the changes between them
