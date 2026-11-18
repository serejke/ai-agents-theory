<a id="flue-against-the-theory-pi--deploy-layer"></a>

# Flue against the theory

Flue is directly on top of the two core packages of PI.
[`@mariozechner/pi-ai`](../pi.md) gives one interface for the LLM and the
Prompt. `@mariozechner/pi-agent-core` gives the AgentLoop with Guardrail hooks.
The [PI case study](../pi.md) compares these parts with the theory. The same
result applies to Flue with no changes. This document tells which parts Flue
adds on top of PI.

<a id="one-line-position"></a>

## The position of Flue

PI applies the rule "primitives, not features" to a coding agent for one user.
Flue gives the same primitives as a unit that you deploy for many tenants.

- PI does not have some parts because of its product. Flue adds these parts:
  Topology, Sessions that stay in storage for many tenants, and an Environment
  for each runtime.
- PI does not have some parts because of a rule. Flue also does not have these
  parts: StateMachine and Memory.

<a id="inherited-from-pi"></a>

## The parts that Flue gets from PI

- The primitives [LLM](../../primitives/llm.md) and
  [Prompt](../../primitives/prompt.md), through `pi-ai`. It has more than 15
  providers and one API that gives the output in parts.
- The [AgentLoop](../../harness/agent-loop.md) as a `while` loop, and the
  [Guardrail](../../harness/guardrail.md) hooks `beforeToolCall` and
  `afterToolCall`, through `pi-agent-core`.
- A [Session](../../harness/session.md) history in one line, with no branches.
- Compaction as a change of the
  [PromptLoading](../../patterns/prompt-loading.md) pattern that operates only
  in the Session.

<a id="added-by-flue"></a>

## The parts that Flue adds

### An HTTP service as the frontend

PI has five [frontends](../../harness/agent.md): an interface in the terminal, a
mode that prints the output, JSON, RPC and SDK. Each of them is around one core
of the agent that runs in one process. The frontend of Flue has a different
structure. It is an HTTP server that the build writes:

- Each agent has an endpoint.
- The file of the agent declares its webhook triggers and the cron entries of
  the manifest.
- The build writes the server from these triggers.

Thus the unit that you deploy is the harness.

### An Environment that you can change: `SessionEnv`

PI uses the local [Environment](../../environment/environment.md): the
filesystem and the processes of the host. Flue adds one interface with `exec`,
`fs`, `cwd` and `scope`. This interface has four adapters:

- a `just-bash` shell in memory
- the filesystem of the host
- `BashFactory`
- external `SandboxFactory` connectors, for example Daytona, Vercel and
  Cloudflare Containers.

In the words of the theory, the Environment becomes a parameter of the Agent. It
is not a property of the host process that stays the same. Thus you can write an
agent one time and deploy it on Node, Cloudflare or GitHub Actions.

### Topology in the framework: `task` subagents

PI does not have [Topology](../../patterns/topology.md), by decision: "no
sub-agents; build with extensions or spawn pi instances via tmux." Flue has it:

- `session.task()` starts a new Session for a subagent. The context of the new
  Session comes from the first Session.
- The maximum depth of these Sessions is 4.
- The LLM can also give a task to a subagent with the `task` tool.

Thus Flue adds a part that PI does not have by decision.

### Skill calls with a result that has a type

In PI, the model selects when to read the `SKILL.md` of a skill. Flue keeps
these skills. It also adds a call that the code of the user starts:
`session.skill(name, { args, result: schema })`. This call gives an output with
a structure, and `valibot` examines the output against the schema.

This call is one more variant of
[PromptLoading](../../patterns/prompt-loading.md). The harness loads the skill
when the user calls it, and the result has a contract with a type. PI does not
give this variant.

### Roles for different agent instructions

The roles are Markdown files in `roles/`. The build puts them into the output.
The frontmatter of each role gives a description and, as an option, a model.
Each role becomes a `<role>...</role>` block of the system prompt.

Flue selects the role in this sequence:

1. the role of the call
2. the role of the Session
3. the role of the agent.

PI has no role concept. In PI, you make roles with extensions.

### Storage that you can change, for many tenants

PI keeps the Sessions as local `.jsonl` files, in a graph of Sessions. The
Sessions of the graph can have the same start. Flue makes the storage a
parameter with `SessionStore`:

- `InMemorySessionStore` on Node
- **Durable Object SQLite** on Cloudflare. A Session continues on the same
  Durable Object across requests, also after some weeks.

The Session ID comes in the path of the request. Thus the Sessions of each
tenant stay apart because of the structure of Flue, not because of a rule for
the user. Thus the [Continuity](../../patterns/continuity.md) pattern can use
this storage. Flue does not supply the Continuity pattern.

### Changes for one call: the scoped runtime

For one call of `prompt`, `skill` or `task`, you can change the tools, the
model, the system prompt, the role and the commands. The scoped runtime keeps a
snapshot of the `Agent` of `pi-agent-core`, does the call and then puts back the
snapshot. PI keeps the data of its Agent as one block. In PI, tools for one call
are possible only with extensions. Flue makes this change a part of its
structure. Thus one Session can use different roles and models on different
turns, and Flue does not make a second copy of the Session.

### The sequence of providers

Flue uses `pi-ai` and adds two functions:

- `provider/model-id` strings
- runtime values for each provider: `baseUrl`, `headers` and `apiKey`.

Flue selects these values in this sequence: the call, then the role, then the
agent. Thus Flue can send all model calls through a server between the agent and
the providers, for example LiteLLM or a server of a company.

### Connectors that an AI agent writes

PI gives new functions as packages, for example `pi install npm:@foo/pi-tools`.
The boundary is an npm package or a git package.

Flue uses a different method. `flue add <provider-url>` sends a Markdown
specification to a coding agent, for example Claude or Codex. The coding agent
writes the `SandboxFactory` adapter into the project of the user. Thus the
contract for extensions is not an API of the runtime. It is a prompt to an
agent.

You do not send a package. You send a specification, and the agent compiles it
into your repository.

<a id="same-gaps-as-pi"></a>

## The parts that Flue and PI do not have

- **No [StateMachine](../../patterns/state-machine.md) and no plan mode.** The
  loop has one phase.
- **No [Memory](../../patterns/memory.md) as a basic part.** `AGENTS.md` is
  [PromptLoading](../../patterns/prompt-loading.md) at the start of the Session,
  from the folder of the Session in the sandbox. It is not Memory in the meaning
  of the theory.
- **No [Continuity](../../patterns/continuity.md).** But you can make it with
  `SessionStore`, Durable Object SQLite and the Markdown files in the sandbox.
