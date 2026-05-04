# AI Agents Theory

A first-principles formalization of what AI agents are made of. The book answers: what is an Agent, what are its parts, and how do they compose?

Most documentation describes agents in terms of specific products and frameworks (Claude Code, Cursor, Codex, LangGraph). The architecture underneath is **universal** — Claude Code, Codex, Cursor, Gemini CLI, PI are different Agents assembled from the same bricks. The model is not the differentiator; the composition is. This repository formalizes those universal building blocks using TypeScript-like pseudocode, so you can design any agent system from scratch — or look at an existing one and immediately see what it uses and what it's missing.

---

## What Is an Agent?

An **Agent** is a program that takes a task, reasons about it with a language model, picks actions (reading files, calling APIs, running shell commands), sees what happened, and keeps going until it's done. You invoke it somehow — on the command line, from another program, through a web request — and results come back through the same channel. When people say "I built an agent that triages my inbox," "the Claude Code agent," or "a Slack agent," they mean one of these runnable programs.

Underneath, every Agent has three parts:

- A **frontend** — how the outside world talks to it and where its output goes. A terminal reads your keystrokes and prints back; an SDK exposes a function call; a GitHub Action fires on an issue and writes a comment. Same Agent underneath, different frontends.
- A **cognitive core** — the reasoning loop. The language model looks at the current situation, picks a tool to run, sees the result, and decides what's next. Turn by turn, the conversation accumulates. The theory calls this the **Session**, and the loop inside it the **[AgentLoop](harness/agent-loop.md)**.
- An **environment** — the substrate it acts on. Your laptop's filesystem, a sandboxed container, a set of API credentials, a database. Two Agents with the exact same cognitive core can behave very differently depending on what's reachable in the environment.

Written as a formula:

```
Agent = Frontend + Session + Environment
```

Everything else in this book is either (a) a piece of an Agent, (b) a way of building an Agent's parts, or (c) a composition that arranges multiple Agents or extends one.

See [Agent](harness/agent.md) for the formal definition, variants, and examples.

---

## The Parts of an Agent

Every Agent decomposes into four kinds of concept, plus verification as an orthogonal concern:

- **Primitives** — the atoms. Irreducible building blocks.
- **Harness** — the machinery that makes an Agent runnable. Agent itself is the outer shell.
- **Environment** — the substrate the Agent acts upon. Not part of the Agent; the world the Agent inhabits.
- **Patterns** — optional arrangements that shape Agent behavior. Some Agents adopt them, others don't.

### Primitives

A primitive is irreducible — it cannot be expressed as a composition of other concepts in the theory. LLM, Tool, and Prompt are the primitives.

| Primitive  | Definition                                                                             | Document                       |
| ---------- | -------------------------------------------------------------------------------------- | ------------------------------ |
| **LLM**    | Stateless function: `string → string` (Prompt in, text out)                            | [LLM](primitives/llm.md)       |
| **Tool**   | Side effect: `(params) → result`                                                       | [Tool](primitives/tool.md)     |
| **Prompt** | The protocol-formatted structured input to the LLM: system + tools + history + inserts | [Prompt](primitives/prompt.md) |

### Harness

The harness is the running-system skeleton — the structural machinery every running agent system has, whether or not the author thought in these terms: iterate the LLM → you have a loop; accumulate turns → you have a session; every tool call flows through a dispatch boundary where policies can intercept → you have a Guardrail; wrap a session in a frontend and pin it to an environment → you have an Agent.

| Harness       | Composition                                            | Document                           |
| ------------- | ------------------------------------------------------ | ---------------------------------- |
| **AgentLoop** | LLM + Tool[] in a recursive cycle                      | [AgentLoop](harness/agent-loop.md) |
| **Session**   | AgentLoop + history                                    | [Session](harness/session.md)      |
| **Guardrail** | Tool-dispatch interception hooks + registered policies | [Guardrail](harness/guardrail.md)  |
| **Agent**     | Frontend + Session + Environment                       | [Agent](harness/agent.md)          |

### Environment

The Environment is what the Agent reaches through Tools — filesystem, processes, network, credentials. An Agent's effective capability is a function of its Environment, especially for generative tools like `bash` where `capability = f(environment)`. Sandboxes are restricted Environments; the Environment layer is where blast-radius control lives.

| Concept         | Shape                                                                | Document                                  |
| --------------- | -------------------------------------------------------------------- | ----------------------------------------- |
| **Environment** | Resource bundle: code, runtime, OS, network, credentials, filesystem | [Environment](environment/environment.md) |

### Patterns

A pattern is a composition of primitives, harness, environment, and other patterns that names a distinct architectural role. Patterns are design choices — some Agents adopt them, others don't. Analogy from functional programming: `map`, `filter`, `reduce` are not new language constructs but patterns over recursion and functions. That doesn't make them less important — patterns are what you use every day.

| Pattern           | Composition                                                           | Document                                    |
| ----------------- | --------------------------------------------------------------------- | ------------------------------------------- |
| **PromptLoading** | `source → strategy → Prompt slot` (eager, lazy, progressive, dynamic) | [PromptLoading](patterns/prompt-loading.md) |
| **Memory**        | Tool-set with cross-session persistence and knowledge-extraction role | [Memory](patterns/memory.md)                |
| **StateMachine**  | Guardrail + phase variable + transition rules                         | [StateMachine](patterns/state-machine.md)   |
| **Channel**       | Tool-set + convention (inter-agent data passing within a run)         | [Channel](patterns/channel.md)              |
| **Planner**       | Session with read-only tools                                          | [Planner](patterns/planner.md)              |
| **Router**        | Dispatch by input                                                     | [Router](patterns/router.md)                |
| **Evaluator**     | LLM-as-judge + feedback loop                                          | [Evaluator](patterns/evaluator.md)          |
| **Workspace**     | Tool-set with hierarchical namespace                                  | [Workspace](patterns/workspace.md)          |
| **Continuity**    | Memory + PromptLoading + Guardrail + Tool                             | [Continuity](patterns/continuity.md)        |
| **Topology**      | Session[] + Channel + Router, arranged in a coordination shape        | [Topology](patterns/topology.md)            |

### Verification

[Verification](verification/index.md) is orthogonal to the tiers above. It doesn't participate in agent reasoning, but every production Agent needs it: how do you confirm the system works, given that the LLM at its core is non-deterministic? Verification defines traces, seams, and strategies for testing Agents end-to-end.

---

## How to Read This

These documents are not numbered chapters. Each is self-contained — you can start with any document that interests you and follow cross-references to deepen understanding.

**If you want to understand what an Agent is:** Start with [Agent](harness/agent.md). Then drill into its parts: [Session](harness/session.md), [AgentLoop](harness/agent-loop.md), [LLM](primitives/llm.md), [Tool](primitives/tool.md), [Environment](environment/environment.md). This gives you the core.

**If you want to understand what's missing from current Agents:** Read [Memory](patterns/memory.md) and [StateMachine](patterns/state-machine.md). Memory is the gap between "starts from zero every session" and "accumulates knowledge over time." StateMachine is the gap between "agent in a terminal" and "agent as a reliable workflow."

**If you want to see the theory applied:** The [LangGraph case study](case-studies/langgraph.md) decomposes a bundled framework into the theory's vocabulary, revealing both what the framework gets right and where it makes constraining trade-offs. The [PI case study](case-studies/pi.md) decomposes the opposite design — a minimal core that exposes every concept boundary as an extension point — and cross-references it against the theory in both directions. The [Hermes Agent case study](case-studies/hermes.md) decomposes a persistent personal-agent framework whose realisation deepens how particular concepts — Prompt cache integrity, layered Memory views, Evaluator-shaped sub-Agents — can be utilised in practice.

---

## Composition Is the Differentiator

The universal framing has an empirical consequence: the same frontier model with different tool designs, context strategies, and guardrail configurations produces dramatically different performance on identical tasks. Two Agents differ not in some holistic, ineffable way, but in specific, enumerable configuration choices: which tools (capped search vs. raw grep), which memory (ground truth feature list vs. none), which guardrails (linter on edit vs. no feedback), which context strategy (progressive disclosure vs. context dump), which environment (laptop vs. sandbox). The composition is where the value lives. The model is commodity.

Claude Code is one specific Agent:

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

Codex, Cursor, Gemini CLI, PI — different Agents from the same bricks. Understanding the parts means understanding all of them.

---

## Who This Is For

Engineers who want to think about Agents from first principles rather than learn a framework. If you've used Claude Code, Cursor, or Codex and wondered "what's actually going on under the hood?" — this gives you the vocabulary to answer that question for any Agent, current or future.

## Status

Living documents. The theory evolves as new Agents emerge and are analyzed.

## License

[MIT](LICENSE)
