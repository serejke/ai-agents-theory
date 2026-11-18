# LLM

An LLM (a large language model, for example Claude or GPT-5) is the part of an
agent system that reads text and writes text. It keeps no data from one call to
the next call. In the theory, the LLM is a function that changes a
[Prompt](prompt.md) into a stream of text. Its input is tokens, and its output
is tokens.

<a id="why-it-matters"></a>

## Value

The LLM is the only concept of an agent system that does reasoning. Each other
concept gives structure to the LLM calls, puts limits on them, or connects them.
It connects them to other LLM calls and to the data and systems around the
agent. These other concepts are:

- the harness: the [AgentLoop](../harness/agent-loop.md), the
  [Session](../harness/session.md), the [Guardrail](../harness/guardrail.md) and
  the [Agent](../harness/agent.md)
- each pattern, for example [Memory](../patterns/memory.md),
  [Channel](../patterns/channel.md) and
  [StateMachine](../patterns/state-machine.md).

Without the LLM, the system cannot think. These facts about the LLM control each
part that uses it:

- The LLM keeps no data between calls.
- At its core, the LLM does not know a protocol. It knows only text.

If you do not know these facts, the structure of the parts around the LLM will
be incorrect.

### The LLM is only a function

The theory shows the LLM as `Prompt → text stream`. It does not show it as
`(system, messages, tools) → (text + tool_calls)`. The second view does not show
the layer where developers make most of their decisions.

The LLM does not know about tools or about the role of a message. It knows the
text patterns of its training. Messages, roles and tool calls are a protocol
layer:

- In the input, the [Prompt](prompt.md) holds the protocol.
- In the output, other code parses the protocol from the stream.

Thus the LLM concept is small: only a function. Then a developer can see the
protocol layer clearly and make decisions about it.

<a id="formal-definition"></a>

## Definition

```typescript
// At the bottom: the LLM is text → text.
type LLM = (input: string) => AsyncStream<string>;

// At the level that code uses: the LLM reads a Prompt and gives a text
// stream. Other code parses the stream to find structured items.
type LLM = (prompt: Prompt) => AsyncStream<string>;

// The protocol layer is in the Prompt. Code applies it in reverse when it
// parses the output stream. This layer makes plain text into tool calls,
// message boundaries and stop tokens. See Prompt.
```

The LLM gives a stream, not one value. It writes its output in small parts, one
after the other. The stream helps in two procedures:

- The user sees the output while the LLM writes it.
- A [Guardrail](../harness/guardrail.md) can examine the output while the LLM
  writes it.

<a id="statelessness-as-an-architectural-constraint"></a>

## The LLM keeps no data between calls

The LLM has no memory between calls and no condition that stays. It knows only
the data of its input. Thus each agent memory is an external part, not a
property of the model.

For example, an Agent "remembers" a decision from a previous Session in these
steps:

1. A [Memory](../patterns/memory.md) system finds the decision.
2. The harness puts the decision into the [Prompt](prompt.md) with
   [PromptLoading](../patterns/prompt-loading.md).
3. The LLM reads the decision in the Prompt.

### The results for an agent system

- **Each call starts again from zero.** Two calls with the same input give
  equivalent outputs, with some random differences. The LLM keeps no data from
  one call for the next call.
- **The Prompt must hold all the context.** If the Agent must know about a
  previous conversation, that conversation must be in the Prompt. The same rule
  applies to the rules of the project.
- **The context has a hard limit.** The context window of the model sets the
  maximum quantity of tokens in the Prompt, for example approximately 200
  thousand tokens. If the text is larger, the Agent cannot see some of the
  information.

### The cause of the theory

The LLM does not keep data across calls. Other primitives and patterns on top of
the LLM do this work. Examples are the decisions of the agent, the knowledge
about the project, and the history of the conversation.

The LLM does not give a memory or a condition across Sessions. It also does not
continue its work after a crash. Most of the other concepts are necessary only
because of these limits. The LLM keeps no data, and users must have systems that
keep data. The theory shows how to make these systems.

<a id="the-protocol-layer"></a>

## The protocol layer

The LLM knows one protocol from its training. The code of the agent must use the
same protocol to serialize the Prompt and to parse the output stream. Different
providers and models use different protocols, for example:

- the tool format of Anthropic
- the function calls of OpenAI
- text with special tokens
- models that use XML in the prompt
- blocks for the reasoning of the model.

The SDK of a provider does this work for the developer. The SDK shows the input
as items with a structure, for example `messages` and `tools`. It shows the
output as items with a structure, for example `ToolCall`. Developers want to
change these items. The SDK does these steps:

1. It serializes the input items into the text format of the protocol.
2. The model operates as `text → text`.
3. It parses the output stream back into items with a structure.

The theory keeps these layers apart:

- The **LLM** is the `text → text` core: the model after its training.
- The **[Prompt](prompt.md)** is the input with a structure. It also includes
  the change of this input into text.
- The [AgentLoop](../harness/agent-loop.md) parses the output text into items
  with a structure.

Thus an Agent can change its provider or its protocol, and the LLM keeps the
same function in the theory. It is always the function that does the reasoning.

## Related

- [Tool](tool.md): the LLM does the reasoning, and the Tool does the operations.
  The Tool is one of the primitives too.
- [Prompt](prompt.md): the input to the LLM. The protocol is in the Prompt.
- [AgentLoop](../harness/agent-loop.md): makes the Prompt, calls the LLM and
  parses the output stream.
- The pattern [PromptLoading](../patterns/prompt-loading.md) puts text from
  different sources into the Prompt.
- [Memory](../patterns/memory.md): how systems keep data on top of an LLM that
  keeps no data.
