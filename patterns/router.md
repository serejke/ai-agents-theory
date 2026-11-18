# Router

A Router selects the agent configuration for each input: the model, the tools or
the target agent that agrees with the task. With this pattern, a small task does
not go to the strongest agent with the highest price when a smaller agent can do
it.

**Composition**: a function from (input, context) to a configuration.

<a id="why-it-matters"></a>

## Value

Different tasks agree with different models and tools:

- A short request about syntax: a fast model with a low price is sufficient.
- A large decision about the structure of a system: use a strong model with a
  high price.
- A task that changes code: the agent must have tools that write.
- An inspection of code: tools that only read are sufficient.

Without a Router, you send all tasks to the strongest agent, and the price is
too high. Or you use one configuration for all tasks, and the quality is lower.

<a id="formal-definition"></a>

## Definition

```typescript
// The Router gets the user input and the other state that the host system
// gives to it (for example the current phase, the user role, the time of day).
// It returns a Session configuration: the tools, the system prompt and the
// LLM of the selected Session.
type Router = (
  input: string,
  context: Record<string, unknown>,
) => {
  tools: Tool[];
  system: string;
  llm: LLM;
};
```

A Router is a usual function. You can write it with rules in code, with an LLM
call, or with text descriptions on the agent definitions. The implementation
controls the price of the Router, how flexible it is, and if its result is
deterministic.

<a id="variants"></a>

## Variants

<a id="heuristic-router"></a>

### Router with rules

A Router with rules uses conditions in code to send each input to a
configuration. It has a low price, and its result is deterministic. But it
breaks easily when an input does not agree with its rules.

```typescript
const heuristicRouter: Router = (input, context) => {
  if (isSimpleQuestion(input)) return { llm: haiku, tools: [fileRead] };
  if (needsCodeChanges(input)) return { llm: sonnet, tools: allTools };
  if (isArchitectural(input)) return { llm: opus, tools: readOnlyTools };
};
```

Claude Code does this in its code, without a Router step. It uses a faster model
for the subagents (the `Task` tool) and a stronger model for the primary loop.

### LLM Router

An LLM Router uses an LLM to find the type of the input and select a
configuration. It is more flexible than a Router with rules. But it has a higher
price, and its result can change for the same input.

```typescript
const llmRouter: Router = async (input) => {
  const decision = await llm("classify", [{ role: "user", content: input }]);
  return configMap[decision]; // "simple" | "code" | "architecture"
};
```

<a id="prompt-as-router"></a>

### Descriptions as a Router

In this variant, the text description of each subagent contains the routing
rules. It occurs in systems where an agent calls other agents as tools (see
[Topology](topology.md#agent-as-tool)).

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
// The orchestrator LLM reads the descriptions and selects the agent to call.
```

The word "AFTER" in the description of `data-analyst` does the work of a guard
in a [StateMachine](state-machine.md). The text descriptions give the conditions
before each agent, the conditions after it, and the limits of each agent.

The good properties of this variant:

- It has no routing code.
- It is flexible: the LLM changes its decisions for each context.
- It is easy to add a new agent.

The bad properties of this variant:

- Its result is not deterministic. The model can ignore an agent, or call agents
  in the incorrect sequence.
- You cannot examine the routing rules.
- It does not make sure that the routing is correct.
- It is not applicable when the agents must operate in a specified sequence.

This variant is applicable to first versions of a system, to systems that you
show to persons, and to systems where approximate routing is sufficient. If a
pipeline for customers must keep a specified sequence of phases, use the phase
changes of a StateMachine.

<a id="handoff--routing-during-execution"></a>

## Handoff: a change of agent during the work

A **handoff** occurs during the work: an agent in the middle of a task gives
control to a different agent at the same level. Usual routing occurs before the
work starts: the Router selects a configuration, and then the work starts.

```typescript
type Handoff = {
  from: string; // the current agent
  to: string; // the target agent
  context: unknown; // the data to give to the target agent
  reason: string; // the cause of the handoff
};
```

In a handoff, the first agent does the task. It finds a part of the task that is
not in its field, and it gives control to a second agent with the related
context. A handoff is different from these operations:

- A call to a subagent: an agent gives a part of its task to a subagent below it
  and gets the result back.
- A Router step before the work: the Router selects the agent before the agent
  starts.

Examples: LangGraph has `Command(goto="other_agent")`. The OpenAI Agents SDK has
a `Handoff` type as a basic part of the SDK.

A handoff is a composition of these parts:

1. **Signal**: the agent that does the work selects to give back control.
   Control moves from the work to the control layer.
2. **Router**: the control layer selects the agent that continues the task.
3. **[Channel](channel.md)**: the context of the first agent moves to the next
   agent.

## Related

- [AgentLoop](../harness/agent-loop.md): the Router configures it and sends the
  input to it.
- [Planner](planner.md): a Router can select one model for the plan and a
  different model for the implementation.
- [StateMachine](state-machine.md): it selects the next phase in code, with
  conditions. Its sequence is deterministic, and the sequence of text
  descriptions in prompts is not.
- [Topology](topology.md): the shapes of the connections between agents. A
  Topology uses Routers to control the flow between agents.
