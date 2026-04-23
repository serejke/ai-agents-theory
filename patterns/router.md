# Router

A Router selects which agent configuration handles a given input — choosing the right model, tools, or target agent based on the task at hand. It is the dispatch layer that prevents every request from going through the most expensive, most capable agent when a simpler one would suffice.

**Composition**: function (input, context) → configuration.

## Why It Matters

Not every task requires the same resources. A simple question about syntax can be handled by a fast, cheap model. A complex architectural decision needs a capable, expensive model. A coding task needs write tools; a review task needs read-only tools. Without routing, you either waste resources (sending everything to the most capable agent) or sacrifice quality (using one configuration for everything).

## Formal Definition

```typescript
// The Router receives the user input and any ambient state the host system
// wants to expose (current phase, user role, time of day, etc.), and returns
// a Session configuration: the tools, system prompt, and LLM the matched
// Session should run with.
type Router = (
  input: string,
  context: Record<string, unknown>,
) => {
  tools: Tool[];
  system: string;
  llm: LLM;
};
```

Router is an ordinary function — it can be implemented as heuristics, as an LLM call, or as natural language descriptions on agent definitions. The implementation determines the trade-off between cost, flexibility, and determinism.

---

## Variants

### Heuristic Router

Rule-based dispatch. Cheap and deterministic but brittle.

```typescript
const heuristicRouter: Router = (input, context) => {
  if (isSimpleQuestion(input)) return { llm: haiku, tools: [fileRead] };
  if (needsCodeChanges(input)) return { llm: sonnet, tools: allTools };
  if (isArchitectural(input)) return { llm: opus, tools: readOnlyTools };
};
```

Claude Code does this implicitly — uses a faster model for sub-agents (Task tool), a more capable model for the main loop.

### LLM Router

Uses an LLM to classify the input and select a configuration. More flexible but expensive and non-deterministic.

```typescript
const llmRouter: Router = async (input) => {
  const decision = await llm("classify", [{ role: "user", content: input }]);
  return configMap[decision]; // "simple" | "code" | "architecture"
};
```

### Prompt-as-Router

In Agent-as-Tool systems, routing logic is encoded as natural language descriptions on each subagent definition:

```typescript
const agents = {
  researcher: {
    description: "Use when you need to gather research information...",
    // ...
  },
  "data-analyst": {
    description: "Use AFTER researchers have completed their work...",
    // ...
  },
};
// The orchestrator LLM reads descriptions and decides which agent to invoke.
```

The word "AFTER" in the data-analyst description does the work of a state transition guard. Preconditions, postconditions, and capability boundaries are encoded in natural language.

**Pros**: Zero routing code. Flexible — LLM adapts to context. Easy to add new agents.
**Cons**: Non-deterministic — model may skip agents or invoke them out of order. Can't audit routing logic. No guarantees. Not suitable when ordering must be strict.

**Applicability**: Prototypes, demos, systems where approximate routing is acceptable. For production pipelines with strict phase ordering, use explicit [StateMachine](state-machine.md) transitions instead.

---

## Handoff — Routing During Execution

Standard routing happens _before_ execution — the router selects a configuration, then execution begins. **Handoff** is routing that happens _during_ execution: an agent mid-task decides to yield control to a peer agent.

```typescript
type Handoff = {
  from: string; // current agent
  to: string; // target agent
  context: unknown; // what to pass along
  reason: string; // why the handoff is happening
};
```

Handoff is different from delegation (which is parent→child, hierarchical) and different from pre-execution routing (which happens before the agent starts). In a handoff, agent A is already working, encounters something outside its expertise, and transfers control to agent B — along with the relevant context.

**Seen in**: LangGraph's `Command(goto="other_agent")`, OpenAI Agents SDK's first-class `Handoff` primitive.

Handoff is a composition of three concerns:

1. **Signal** — the executing agent decides it needs to yield (from execution to orchestration)
2. **Router** — the orchestration layer selects which agent takes over
3. **Channel** — the context from the current agent transfers to the next

---

## Related

- [AgentLoop](../harness/agent-loop.md) — what the Router configures and dispatches to
- [Planner](planner.md) — Router may select different models for planning vs. implementation
- [StateMachine](state-machine.md) — code-level routing with guards; more reliable than prompt-based routing
- [Topology](topology.md) — multi-agent coordination shapes that use routing for control flow between agents
