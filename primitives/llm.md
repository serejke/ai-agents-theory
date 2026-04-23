# LLM

The LLM is a stateless function that maps a [Prompt](prompt.md) to a stream of text. At the bottom, it's `string → string` — tokens in, tokens out. Every other appearance of the LLM (structured messages, tool calls, role tags) is a protocol layer that lives in the Prompt on the way in and is parsed out of the output stream on the way back.

## Why It Matters

The LLM is the only concept in agent systems that performs reasoning. The harness ([AgentLoop](../harness/agent-loop.md), [Session](../harness/session.md), [Guardrail](../harness/guardrail.md), [Agent](../harness/agent.md)) and every pattern ([Memory](../patterns/memory.md), [Channel](../patterns/channel.md), [StateMachine](../patterns/state-machine.md), and the rest) exist to structure LLM calls, constrain them, or connect them to other LLM calls and to the world. Without the LLM, the system has no intelligence. Without understanding that the LLM is _stateless_ and _protocol-agnostic at its core_, you will misdesign everything that depends on it.

Treating the LLM as `Prompt → text stream` rather than as `(system, messages, tools) → (text + tool_calls)` matters because the latter hides the layer where most design happens. The LLM doesn't natively know about "tools" or "roles" — it recognizes text patterns it was trained to recognize. Everything structured lives in the [Prompt](prompt.md) (input) or gets parsed from the output stream by downstream code. Keeping the LLM primitive narrow — just a function — makes the protocol layer visible and design-able.

## Formal Definition

```typescript
// At the bottom: LLM is text → text.
type LLM = (input: string) => AsyncStream<string>;

// At the usable level: LLM consumes a Prompt and produces a text stream
// that downstream code can parse for structured elements.
type LLM = (prompt: Prompt) => AsyncStream<string>;

// The protocol layer — baked into the Prompt and applied in reverse when
// parsing the output stream — is what turns plain text into structured
// tool calls, message boundaries, and stop tokens. See Prompt.
```

The LLM has no memory between calls. It has no state. Everything it "remembers" is what was passed in its input. This is fundamental: any agent "memory" is an external construct, not a property of the model. When an Agent "remembers" a decision from yesterday, that knowledge was retrieved from a [Memory](../patterns/memory.md) system, injected into the [Prompt](prompt.md) by a [PromptLoading](../patterns/prompt-loading.md) strategy, and only then read by the LLM.

The return type is a stream, not a single value. The LLM produces output incrementally. The streaming property matters for user experience (progressive display) and for [Guardrails](../harness/guardrail.md) (which can inspect output as it's generated).

---

## Statelessness as an Architectural Constraint

Statelessness means:

- **No learning between calls.** Two identical calls with identical inputs produce (probabilistically) equivalent outputs. The LLM cannot "learn" from one call to the next.
- **All context must be explicit.** If the Agent needs to know about a previous conversation, that conversation must be in the Prompt. If it needs to know project conventions, those must be in the Prompt.
- **Context has a hard ceiling.** The Prompt is bounded by the model's context window (~200K tokens for current models). When content overflows, information is lost — the Agent effectively forgets.

Every persistent capability of an agent system — remembering decisions, accumulating project knowledge, maintaining conversation history — is built _on top of_ this stateless function by other primitives and patterns. The LLM itself provides none of it.

This constraint is what makes the entire theory necessary. If the LLM had built-in persistent memory, cross-session state, and durable execution, most of the other concepts would be redundant. The theory exists because the LLM is stateless and the world needs stateful systems.

---

## The Protocol Layer

Different providers and different models speak different protocols: Anthropic's tool-use format, OpenAI's function calling, raw text with special tokens, XML-prompted models, reasoning blocks, etc. The LLM is trained to recognize a specific protocol; agent code must speak that same protocol when serializing the Prompt, and must use the same protocol when parsing the output stream.

Think of provider SDKs as convenient wrappers: they present structured inputs (`messages`, `tools`) and structured outputs (`ToolCall` events) as first-class, because that's what developers want to manipulate. Under the hood, the SDK serializes structured input into the protocol's text format, the model runs as `text → text`, and the SDK parses the output stream back into structured events.

The theory separates these layers:

- The **LLM** is the `text → text` core — the trained model.
- The **[Prompt](prompt.md)** is the structured input and its serialization to text.
- The **output parsing** (text → structured events) happens at the [AgentLoop](../harness/agent-loop.md) boundary.

This separation lets an Agent swap providers or protocols without changing the LLM's role in the theory: it's always "the reasoning function."

---

## Related

- [Tool](tool.md) — the other primitive; LLM reasons, Tool acts
- [Prompt](prompt.md) — the structured input to the LLM; the protocol lives here
- [AgentLoop](../harness/agent-loop.md) — assembles the Prompt, calls the LLM, parses the output stream
- [PromptLoading](../patterns/prompt-loading.md) — strategies for filling the Prompt
- [Memory](../patterns/memory.md) — how persistence is built on top of statelessness
