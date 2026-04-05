# Tool

A Tool is a named function with a schema that performs a side effect and returns a result. It is the only primitive through which an agent affects the outside world.

## Why It Matters

The [LLM](llm.md) reasons but cannot act. It cannot read a file, query a database, send an email, or execute code. Every interaction between an agent and its environment goes through Tools. The set of Tools available to an agent defines what the agent can do — an agent with `[fileRead, fileWrite, bash]` is a coding assistant; the same agent with `[queryDatabase, sendEmail, createTicket]` is a support agent. The architecture is identical; the capabilities differ entirely.

## Formal Definition

```typescript
type Tool = {
  name: string;
  schema: JSONSchema; // input schema — what the LLM sees when deciding to call
  execute: (params: unknown) => Promise<ToolResult>; // side effect!
};

type ToolResult = {
  output: string;
  error?: string;
};
```

The `schema` serves a dual purpose: it tells the LLM what parameters the tool accepts (for correct invocation), and it tells the LLM what the tool _does_ (via the name and description fields in the schema). The LLM reads the schema to decide whether and how to call the tool.

The `execute` function is the only place in the entire agent system where side effects occur. Everything else — LLM calls, routing decisions, memory retrieval — is either pure computation or reads. Tools write to the world.

---

## Tools as the Architectural Boundary

Tools define the boundary between agent primitives and regular software. The `execute(params) → result` interface is the seam:

```
Agent primitives:   LLM + Tools + Memory + Guardrails + StateMachine + Channel
                           ↓
Tool boundary:      execute(params) → result
                           ↓
Everything below:   Regular software (databases, pipelines, APIs, infrastructure)
```

- **Above the boundary**: agent primitives — LLM, Memory, Guardrails, Channels, orchestration. This is where non-deterministic reasoning happens.
- **Below the boundary**: regular software engineering — databases, APIs, pipelines, infrastructure. This is deterministic, well-understood, and testable with conventional techniques.

The agent doesn't know or care what lives behind the tool interface. A `query_portfolio` tool might execute a simple SQL query or trigger a multi-stage ETL pipeline — the agent sees the same `(params) → result` signature either way. This separation is what makes agent systems composable: you can swap the implementation behind any tool without changing the agent.

This boundary is central to [App Inversion](../app-inversion/architecture.md) — the thesis that applications decompose into tool-shaped capabilities consumed by agents.

---

## Related

- [LLM](llm.md) — the other atomic primitive; LLM reasons, Tool acts
- [AgentLoop](../patterns/agent-loop.md) — the recursive cycle that connects LLM decisions to Tool execution
- [Guardrail](guardrail.md) — constraints that wrap tool execution with pre/post checks
- [App Inversion](../app-inversion/architecture.md) — how the tool boundary reshapes software architecture
