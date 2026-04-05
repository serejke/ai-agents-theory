# LLM

The LLM is a stateless function that maps a system prompt and a sequence of messages to a stream of text and tool calls. It is the core computation primitive of every agent system.

## Why It Matters

The LLM is the only primitive that performs reasoning. Every other primitive — [Tool](tool.md), [Memory](memory.md), [Guardrail](guardrail.md), [Channel](channel.md), [StateMachine](state-machine.md) — exists to provide structure around LLM calls, constrain them, or connect them. Without the LLM, the system has no intelligence. Without understanding that the LLM is _stateless_, you will misdesign everything that depends on it.

## Formal Definition

```typescript
// Message — unit of communication
type Message =
  | { role: "user"; content: string }
  | { role: "assistant"; content: string; toolCalls?: ToolCall[] }
  | { role: "tool_result"; callId: string; result: string };

// LLM — pure function (stateless). All state is in the arguments.
type LLM = (
  system: string,
  messages: Message[],
) => AsyncStream<TextDelta | ToolCall>;
```

The LLM has no memory between calls. It has no state. Everything it "remembers" is what was passed in `messages`. This is fundamental: any agent "memory" is an external construct, not a property of the model. When an agent "remembers" a decision from yesterday, that knowledge was retrieved from a [Memory](memory.md) system and injected into `messages` before the LLM call.

The `system` parameter defines the agent's identity, capabilities, and constraints. Two agents using the same model but different system prompts are fundamentally different agents — the system prompt is the primary configuration surface.

The return type is a stream, not a single value. The LLM produces output incrementally: text tokens for the user and tool calls for the [AgentLoop](../patterns/agent-loop.md) to execute. The streaming property matters for user experience (progressive display) and for [Guardrails](guardrail.md) (which can inspect output as it's generated).

---

## Statelessness as an Architectural Constraint

Statelessness means:

- **No learning between calls.** Two identical calls with identical arguments produce (probabilistically) equivalent outputs. The LLM cannot "learn" from one call to the next.
- **All context must be explicit.** If the agent needs to know about a previous conversation, that conversation must be in `messages`. If it needs to know project conventions, those must be in `system`.
- **Context has a hard ceiling.** The `messages` array is bounded by the model's context window (~200K tokens for current models). When context overflows, information is lost — the agent effectively forgets.

Every persistent capability of an agent system — remembering decisions, accumulating project knowledge, maintaining conversation history — is built _on top of_ this stateless function by other primitives and patterns. The LLM itself provides none of it.

This constraint is what makes the entire theory necessary. If the LLM had built-in persistent memory, cross-session state, and durable execution, most of the other primitives would be redundant. The theory exists because the LLM is stateless and the world needs stateful systems.

---

## Related

- [Tool](tool.md) — the other atomic primitive; LLM reasons, Tool acts
- [AgentLoop](../patterns/agent-loop.md) — the first composition of LLM + Tool
- [ContextProvider](../patterns/context.md) — how knowledge is injected into the LLM's `system` and `messages`
- [Memory](memory.md) — how persistence is built on top of statelessness
