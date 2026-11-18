# AI Agents Theory

This book tells which parts make an AI agent and how these parts connect. It
does not start from a product. It starts from the basic functions that each
agent must have. The book tells:

- the definition of an Agent
- which parts an Agent has
- how these parts compose.

Most documents about agents give one product or one framework, for example
Claude Code, Cursor, Codex or LangGraph. But the structure below these products
is the same for all of them. Claude Code, Codex, Cursor, Gemini CLI and PI are
different Agents that are made of the same parts. The model does not make one
Agent different from a different Agent. The composition does: the parts and how
they connect.

This book gives each part as a short type in TypeScript pseudocode. With these
types, you can build an agent system from zero. You can also examine an agent
system, and see quickly which parts it uses and which parts it does not have.

---

<a id="what-is-an-agent"></a>

## The Agent

An **Agent** is a program that gets a task and uses a language model to think
about the task. The model selects steps, for example: read files, call APIs or
run shell commands. The Agent examines the result of each step and continues
until it completes the task.

You start an Agent from the command line, from a different program or with a web
request. The results come back through the same path. A person can speak about
"an agent that triages my inbox", "the Claude Code agent" or "a Slack agent".
Each of these is a program of this type.

Each Agent has these parts:

- A **frontend**: the connection between the Agent and its users or other
  programs. The frontend gets the input and sends the output. A terminal reads
  the keys that you push and shows the output. An SDK gives a function that a
  program calls. A GitHub workflow starts when a person opens an issue, and then
  writes a comment. The Agent is the same in each example. Only the frontend is
  different.
- A **reasoning core**: the loop of the model. The model examines the condition
  of the task, selects a tool to run, reads the result and selects the next
  step. Each turn adds to the conversation. The theory calls this part the
  **Session**, and the loop in it the **[AgentLoop](harness/agent-loop.md)**.
- An **environment**: the files, programs and data that the Agent changes.
  Examples are the filesystem of your computer, a container with a sandbox, a
  set of API credentials and a database. Two Agents with the same reasoning core
  can operate very differently. The cause is the resources that each environment
  makes available.

The composition of an Agent:

```
Agent = Frontend + Session + Environment
```

Each other concept of this book is one of these:

- a part of an Agent
- a method to build the parts of an Agent
- a composition that connects two or more Agents, or that adds functions to one
  Agent.

[Agent](harness/agent.md) gives the full definition, the variants and the
examples.

---

## The Parts of an Agent

The concepts of an Agent are in these tiers:

- **Primitives**: the smallest parts. The theory cannot make them from other
  parts.
- **Harness**: the parts that let an Agent run. The Agent is the outer part of
  the harness.
- **Environment**: the files, programs and data that the Agent changes. It is
  not a part of the Agent.
- **Patterns**: optional structures that change how an Agent operates. Some
  Agents use them, and other Agents do not.

Verification is not a tier. It applies to all the tiers.

### Primitives

The primitives are the concepts that the theory cannot make from other concepts.
LLM, Tool and Prompt are the primitives.

| Primitive  | Definition                                                                                        | Document                       |
| ---------- | ------------------------------------------------------------------------------------------------- | ------------------------------ |
| **LLM**    | A function that keeps no data between calls: `string → string`. A Prompt goes in, text comes out. | [LLM](primitives/llm.md)       |
| **Tool**   | An operation with a side effect: `(params) → result`.                                             | [Tool](primitives/tool.md)     |
| **Prompt** | The input of the LLM in the format of its training: system prompt, tools, history, inserts        | [Prompt](primitives/prompt.md) |

### Harness

The harness is the structure that each agent system has when it runs, also if
its author does not use these names:

- If you call the LLM again and again, you have a loop.
- If you keep the turns, you have a Session.
- Each tool call goes through one point before the tool runs, and policies can
  stop the call there. This point is the Guardrail.
- If you put a Session in a frontend and connect it to an environment, you have
  an Agent.

| Harness       | Composition                                                | Document                           |
| ------------- | ---------------------------------------------------------- | ---------------------------------- |
| **AgentLoop** | LLM + Tool[] in a loop that starts again after each result | [AgentLoop](harness/agent-loop.md) |
| **Session**   | AgentLoop + history                                        | [Session](harness/session.md)      |
| **Guardrail** | Hooks that stop each tool call + the policies of the hooks | [Guardrail](harness/guardrail.md)  |
| **Agent**     | Frontend + Session + Environment                           | [Agent](harness/agent.md)          |

### Environment

The Environment is the set of resources that the Agent can use with its Tools:
the filesystem, the processes, the network and the credentials. The Environment
controls the functions of an Agent. This is most important for a tool that can
do almost each operation, for example `bash`. For such a tool,
`capability = f(environment)`. A sandbox is an Environment with limits. The
Environment is the layer that makes the damage of an incorrect step smaller.

| Concept         | Shape                                                                                 | Document                                  |
| --------------- | ------------------------------------------------------------------------------------- | ----------------------------------------- |
| **Environment** | A set of resources: code, runtime, operating system, network, credentials, filesystem | [Environment](environment/environment.md) |

### Patterns

A pattern is a composition of primitives, harness concepts, the Environment and
other patterns. It gives a name to one function in the structure of a system.
Patterns are decisions about the structure: some Agents use them, and other
Agents do not.

An example from code: `map`, `filter` and `reduce` are not new parts of a
language. They are patterns of functions and of a function that calls the same
function again. They are important all the same: you use patterns all the time.

| Pattern           | Composition                                                                       | Document                                    |
| ----------------- | --------------------------------------------------------------------------------- | ------------------------------------------- |
| **PromptLoading** | `source → strategy → Prompt slot` (at the start, on request, in steps, each turn) | [PromptLoading](patterns/prompt-loading.md) |
| **Memory**        | Tools that keep knowledge across Sessions and find the knowledge to keep          | [Memory](patterns/memory.md)                |
| **StateMachine**  | Guardrail + phase variable + rules that change the phase                          | [StateMachine](patterns/state-machine.md)   |
| **Channel**       | Tools + an agreed format that moves data between agents in one run                | [Channel](patterns/channel.md)              |
| **Planner**       | Session with only tools that read                                                 | [Planner](patterns/planner.md)              |
| **Router**        | Selection of a handler by the input                                               | [Router](patterns/router.md)                |
| **Evaluator**     | LLM that examines the output + feedback loop                                      | [Evaluator](patterns/evaluator.md)          |
| **Workspace**     | Tools on a tree of items with names                                               | [Workspace](patterns/workspace.md)          |
| **Continuity**    | Memory + PromptLoading + Guardrail + Tool                                         | [Continuity](patterns/continuity.md)        |
| **Topology**      | Session[] + Channel + Router, connected in a shape                                | [Topology](patterns/topology.md)            |

### Verification

[Verification](verification/index.md) applies to all the tiers. It is not a part
of the reasoning of an agent, but it is necessary for each Agent that customers
use. The LLM in an Agent can give different outputs for the same input.
Verification tells how to show that the system operates correctly all the same.
It gives traces, seams and methods for tests of Agents from end to end.

---

## How to Read This

The documents do not have a sequence. You can read each document without the
other documents. Start with the document that you want, and then follow the
links to know more.

**For the core of an Agent:** Start with [Agent](harness/agent.md). Then read
about its parts: [Session](harness/session.md),
[AgentLoop](harness/agent-loop.md), [LLM](primitives/llm.md),
[Tool](primitives/tool.md) and [Environment](environment/environment.md).

**For an Agent that keeps knowledge and follows a workflow:** Read
[Memory](patterns/memory.md) and [StateMachine](patterns/state-machine.md).

- Memory makes the difference between an agent that starts with no knowledge in
  each Session and an agent that collects knowledge across Sessions.
- StateMachine makes the difference between an agent in a terminal and an agent
  that follows a workflow correctly each time.

**For the theory on products:** Read the case studies.

- The [LangGraph case study](case-studies/langgraph.md) divides a framework with
  many parts in one package into the concepts of the theory. It shows the good
  decisions of the framework, and the decisions that put limits on its users.
- The [PI case study](case-studies/pi.md) examines the opposite structure: a
  small core that lets a developer add code at the boundary of each concept. It
  compares PI with the theory, and the theory with PI.
- The [Hermes Agent case study](case-studies/hermes.md) examines a framework for
  an agent of one person that operates all the time. It shows how some concepts
  operate in a product:
  - a cache of the Prompt that stays correct
  - Memory in layers
  - Agents that a different Agent starts, with the shape of an Evaluator.
- The [Flue case study](case-studies/flue/README.md) examines a framework that
  runs agents in sandboxes on servers.
- The [Honcho case study](case-studies/honcho/README.md) examines a Memory
  service for agents.

**For security:** Read [Security](security/README.md). It is not a tier. It
examines the dangerous results that an agent can cause. It also shows how the
Environment, the Guardrail, the Tool and the Topology stop these results.

---

<a id="composition-is-the-differentiator"></a>

## The Composition Makes the Difference

All Agents are made of the same parts. Thus the parts and their configuration
control the result. The same model of the highest quality gives very different
results on the same tasks with different tools, context methods and Guardrail
configurations.

Two Agents are different in a list of specified configuration decisions:

- tools: a tool that finds text and gives a maximum number of results, or `grep`
  with no limit
- Memory: a correct list of features, or no Memory
- Guardrails: a linter after each change, or no feedback
- the context method: the context in steps, or all the context at the start
- Environment: your computer or a sandbox.

The value is in the composition. Many companies supply a model of the same
quality.

Claude Code is one specified Agent:

```typescript
const claudeCode: Agent = {
  frontend: { kind: "cli", interactive: true },
  session: Session(AgentLoop(claude_sonnet, codeTools, systemPrompt)),
  environment: { code: localFS, runtime: localNode, os: localShell },
  memory: claudeMd, // manual, always_load
  guardrails: [noForcePush, askBeforeWrite, safetyRules],
  stateMachine: twoPhasePlanMode, // plan -> implement (degenerate)
};
```

Codex, Cursor, Gemini CLI and PI are different Agents with the same parts. If
you know the parts, you know all of these Agents.

---

## Who This Is For

This book is for engineers who want to think about Agents from the basic facts,
and not from one framework. You possibly used Claude Code, Cursor or Codex, and
wanted to know "what's actually going on under the hood?" This book gives you
the words to tell how each Agent operates. The words apply to the Agents that
persons use, and to the Agents that persons will make.

<a id="status"></a>

## Changes

The documents change. The theory changes when persons make new Agents and the
author examines them.

## License

The [LICENSE](LICENSE) file gives the "MIT License".
