# AI Agents Theory

Every agent system — whether it's a CLI tool, a GitHub bot, or a multi-agent research pipeline — is assembled from the same small set of building blocks. But most documentation describes agents in terms of specific frameworks (LangChain, CrewAI, AutoGen) rather than the universal primitives underneath.

This repository formalizes those building blocks using TypeScript-like pseudocode. The goal: if you understand the primitives, you can design any agent system from scratch — or look at any existing one and immediately see which primitives it uses and which it's missing.

---

## The Taxonomy

Agent systems decompose into three categories: **primitives** (irreducible building blocks), **patterns** (compositions of primitives), and **infrastructure** (deployment and operational concerns).

### Primitives

A primitive is irreducible — it cannot be expressed as a composition of other primitives. Remove any primitive and a class of agent systems becomes impossible to build.

| Primitive        | Definition                                        | Document                                    |
| ---------------- | ------------------------------------------------- | ------------------------------------------- |
| **LLM**          | Stateless function: `(system, messages) → stream` | [LLM](primitives/llm.md)                    |
| **Tool**         | Side effect: `(params) → result`                  | [Tool](primitives/tool.md)                  |
| **Memory**       | Persistent read+write context across sessions     | [Memory](primitives/memory.md)              |
| **Guardrail**    | Hard constraint not controlled by the LLM         | [Guardrail](primitives/guardrail.md)        |
| **StateMachine** | Phases + transitions + human checkpoints          | [StateMachine](primitives/state-machine.md) |
| **Channel**      | Intermediate results between agents within a run  | [Channel](primitives/channel.md)            |

### Patterns

A pattern is a composition of primitives that recurs across agent systems. Patterns are not new building blocks — they are standard ways of assembling existing ones. Analogy from functional programming: `map`, `filter`, `reduce` are not new language constructs but patterns over recursion and functions. That doesn't make them less important — patterns are what you use every day.

| Pattern             | Composition                                  | Document                               |
| ------------------- | -------------------------------------------- | -------------------------------------- |
| **AgentLoop**       | LLM + Tool[] in a recursive cycle            | [AgentLoop](patterns/agent-loop.md)    |
| **Session**         | AgentLoop + history                          | [Session](patterns/session.md)         |
| **ContextProvider** | `() → string` — knowledge injection into LLM | [ContextProvider](patterns/context.md) |
| **Planner**         | Session with read-only tools                 | [Planner](patterns/planner.md)         |
| **Router**          | Dispatch by input                            | [Router](patterns/router.md)           |
| **Evaluator**       | LLM-as-judge + feedback loop                 | [Evaluator](patterns/evaluator.md)     |
| **Workspace**       | Tool + Channel + ContextProvider             | [Workspace](patterns/workspace.md)     |
| **Deployment**      | Trigger + Session + Output + Environment     | [Deployment](patterns/deployment.md)   |

### Infrastructure

Infrastructure concerns that don't participate in agent reasoning but are required for a working system.

| Concern          | Role                                                               |
| ---------------- | ------------------------------------------------------------------ |
| **Trigger**      | What starts the agent (CLI, issue, cron, webhook)                  |
| **Output**       | Where the agent writes (stdout, PR, comment, file)                 |
| **Environment**  | What the agent has access to (code, runtime, OS)                   |
| **Verification** | How you confirm the system works — [Verification](verification.md) |

---

## How to Read This

These documents are not numbered chapters. Each is self-contained — you can start with any document that interests you and follow cross-references to deepen understanding.

**If you want to understand what agent systems are made of:** Start with [LLM](primitives/llm.md) and [Tool](primitives/tool.md), then [AgentLoop](patterns/agent-loop.md) and [Session](patterns/session.md). This gives you the core: a stateless function + side effects + a recursive loop + conversation history. Everything else builds on this foundation.

**If you want to understand what's missing from current systems:** Read [Memory](primitives/memory.md) and [StateMachine](primitives/state-machine.md). Memory is the gap between "starts from zero every session" and "accumulates knowledge over time." StateMachine is the gap between "agent in a terminal" and "agent as a reliable workflow."

**If you want to understand how software changes:** Read [App Inversion](app-inversion/architecture.md) and its [Economics](app-inversion/economics.md). These analyze which parts of traditional applications survive, which get replaced, and what happens to markets when apps become agent capabilities.

**If you want to see the theory applied:** The [LangGraph case study](case-studies/langgraph.md) decomposes a real framework into the theory's vocabulary, revealing both what the framework gets right and where it makes constraining trade-offs.

---

## Universality

All primitives and patterns are **universal** — none are specific to any particular tool or framework. Claude Code, Codex, Devin, Gemini CLI, Cursor — different assemblies from the same bricks. The difference is in specific parameter values, not architecture.

Claude Code is one specific assembly:

```typescript
const claudeCode: AgentDeployment = {
  trigger: "cli_stdin",
  session: Session(AgentLoop(claude_sonnet, codeTools, systemPrompt)),
  output: "cli_stdout",
  environment: { code: localFS, runtime: localNode, os: localShell },
  memory: claudeMd, // manual, always_load
  guardrails: [noForcePush, askBeforeWrite, safetyRules],
  stateMachine: twoPhasePlanMode, // plan -> implement (degenerate)
};
```

Codex, Devin, Gemini CLI, Cursor — different assemblies from the same bricks. Understanding the primitives means understanding all of them.
