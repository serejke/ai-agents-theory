# Prompt

A Prompt is the text input that the [LLM](llm.md) reads in one call. It holds
the instructions of the agent, the tool schemas, the messages of this turn and
the inserts. Code serializes all of these parts into text in the protocol of the
provider. Examples are the tool format of Anthropic, the chat format of OpenAI,
text with special tokens, and XML.

The Prompt changes the condition of the agent into text that the LLM can read.
The messages of the Prompt are a snapshot of the
[Session](../harness/session.md) history and the new input of the user. The
Prompt is a view for one call, not a store of data. The Session keeps the
history, and the Prompt borrows it for one call.

<a id="why-it-matters"></a>

## Value

The Prompt makes an LLM operate as an assistant that uses tools. At its core,
the [LLM](llm.md) is `string → string`. It does not know "tools", "roles" or
"tool calls". It knows only tokens. The Prompt gives its text a structure in a
protocol, and the model knows this protocol from its training. These parts of
the protocol are in the Prompt, not in the LLM:

- the role of each message, for example a message from the user or from the
  assistant
- the JSON blocks that tell the LLM about a tool that it can call
- the tokens that tell the LLM to stop at the end of its turn.

### The point where the concepts go into the LLM

The other concepts of the theory go into the LLM through the Prompt. All of them
change into the same shape before the LLM runs:

- The history of the [Session](../harness/session.md) becomes the messages of
  the Prompt.
- The definitions of each [Tool](tool.md) become the tool schemas of the Prompt.
- The instructions of the agent become the system prompt.
- With [PromptLoading](../patterns/prompt-loading.md), text from other sources
  goes into the Prompt. Examples are files, facts from
  [Memory](../patterns/memory.md) and skills.

### Slots, not a composition

No other concept of the theory gives the structure of the Prompt. Thus the
Prompt is one of the primitives. The training of the LLM gives this structure.
The concepts of the theory fill the slots of the Prompt. They do not compose the
Prompt.

For example, two agents can have the same Session and the same Tools, but use
different protocols of different providers. Then their Prompts have different
shapes.

<a id="formal-definition"></a>

## Definition

```typescript
// The shape that agent code uses. The serialization of the protocol makes
// the string that the LLM reads.
type Prompt = {
  system?: string; // the agent's identity, instructions and context that is always loaded
  tools?: ToolSchema[]; // the tools that the model can call
  messages: Message[]; // a snapshot of Session.history for this call, plus new user input
  inserts?: string[]; // text added for one turn (found documents, tool results)
};

type Message =
  | { role: "user"; content: string }
  | { role: "assistant"; content: string; toolCalls?: ToolCall[] }
  | { role: "tool_result"; callId: string; result: string };

type ToolCall = { id: string; name: string; args: Record<string, unknown> };

type ToolSchema = {
  name: string;
  description: string; // the LLM reads this to decide if and when to call the tool
  input: JSONSchema; // JSON Schema: a standard format, not a theory concept
};

// Serialization: the LLM provider defines it for each protocol.
type Protocol = "anthropic" | "openai" | "xml" | "raw-text";

const serialize = (prompt: Prompt, protocol: Protocol): string => {
  /* ... */
};
```

The input of the LLM is always a string or a sequence of tokens. The agent code
changes the Prompt as items with a structure. The protocol serializes these
items into the last string. Usually the SDK of the provider does this work. You
give it the messages and the tools, and the SDK puts them into the format of the
model.

<a id="prompt-vs-llm"></a>

## The Prompt and the LLM

The Prompt and the LLM are two different concepts. Keep them apart:

| Concept    | Function                                                                                                   |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| **LLM**    | A function that keeps no data between calls. It reads a Prompt and gives a stream of output text.          |
| **Prompt** | The input with a structure: the system prompt, the tools, the history and the inserts, in a protocol text. |

The LLM does not know the structure of its input or of its output. The structure
is in the Prompt, or other code parses it from the output text. The theory has
no concept for the output. The output is only text. The
[AgentLoop](../harness/agent-loop.md) reads this text and divides the last text
for the user from the tool calls. It uses the same protocol that the Prompt used
for the input.

<a id="what-goes-in-a-prompt"></a>

## The parts of a Prompt

The harness makes a new Prompt for each call to the LLM. Each part comes from a
different source:

| Part                 | Source                                                                                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| System prompt        | The configuration of the agent. Frequently it comes from two or more sources, for example `CLAUDE.md`, the role of the agent and the instructions of a phase. |
| Tool schemas         | The Tools that the Agent gives to the LLM at this time.                                                                                                       |
| Messages of the turn | A snapshot of the [Session](../harness/session.md) history and the new input of the user.                                                                     |
| Inserts              | The text for one turn from [PromptLoading](../patterns/prompt-loading.md), for example facts from Memory, the text of a skill, or tool results.               |

The developer selects the source that fills each slot. The pattern
[PromptLoading](../patterns/prompt-loading.md) gives these decisions:

- At the start: `CLAUDE.md` and the structure of the project go into `system`.
- On request: a skill goes into `messages` or `system` only when the agent must
  have it.
- For each turn: facts from Memory and tool results come new for each call.

<a id="protocol-as-the-contract-between-llm-and-prompt"></a>

## The protocol between the LLM and the Prompt

The training of the LLM sets the protocol that it knows. The Prompt must use
this protocol when it changes into text. Different providers use different
protocols:

- **Anthropic**: the API has `system`, `messages[]` and `tools[]`. A tool call
  comes back in the output as a `tool_use` block.
- **OpenAI**: `messages[]` has the roles `system`, `user`, `assistant` and
  `tool`. The function calls use `tools[]`. A tool call comes back in the
  `tool_calls` field.
- **One string with special tokens**: some systems of a previous type put all
  the parts into one string. Special tokens, for example `<|im_start|>`, divide
  the parts.
- **XML**: some models give the best results with XML in a string, for example
  `<task>` and `<tool_call>`.

You can run the same Session with two different LLMs. Then each LLM can have a
different text of the Prompt. This difference is in the Prompt, not in the
Session.

<a id="prompt-and-tool-definitions"></a>

## Tool schemas in the Prompt

A tool schema in the Prompt is not the Tool. The Tool is a function in the code
of the agent: `execute(params) → result`. The Prompt holds only the schema of
the Tool: its name, its description and the shape of its input. With the schema,
the LLM knows that the Tool is available and how to call it.

If the output of the LLM has a tool call, the code of the agent finds the Tool
with that name. Then it runs the `execute` function of that Tool. The Prompt
tells the LLM which Tools it can use. The [AgentLoop](../harness/agent-loop.md)
runs the Tool that the LLM selects.

The bash tool shows this difference clearly. Its schema in the Prompt has one
input: `command: string`. But the [Environment](../environment/environment.md)
sets the operations that the bash tool can do, and these operations are very
many. The tool schema is small, but the Tool can do many operations.

## Related

- [LLM](llm.md): the function that reads the Prompt.
- [Tool](tool.md): the item that each tool schema of the Prompt identifies.
- [Session](../harness/session.md): the source of the history of the Prompt.
- The pattern [PromptLoading](../patterns/prompt-loading.md) fills the slots of
  the Prompt at the start, on request, in steps or for each turn.
- [AgentLoop](../harness/agent-loop.md): makes the Prompt for each loop and
  parses the output stream of the LLM.
