# Prompt

A Prompt is the structured text input fed into the [LLM](llm.md) for a single call. It carries system instructions, tool schemas, the messages the LLM should see this turn (a snapshot of [Session](../harness/session.md) history plus any pending user input), and any dynamic inserts — all serialized per a provider-specific protocol (Anthropic tool use, OpenAI chat format, raw text, XML tags). The Prompt is what turns "agent state" into something the LLM can actually read.

The Prompt is a per-call _view_, not a persistent store. Session owns the conversation history; Prompt borrows it for one call.

## Why It Matters

The [LLM](llm.md) is fundamentally `string → string`. It doesn't natively understand "tools," "roles," or "tool calls" — it understands tokens. What makes an LLM behave as a tool-using assistant is the Prompt: the text is structured according to a protocol the model was post-trained to recognize. The role tags that tell the LLM "this is a user message vs. assistant message," the JSON blocks that tell it "this is a callable tool," the stop tokens that tell it "end of turn" — all of this is Prompt, not LLM.

This matters because the Prompt is where the theory's other concepts cross into the LLM's world. [Session](../harness/session.md) history becomes Prompt messages. [Tool](tool.md) definitions become Prompt tool schemas. System instructions become the Prompt's system block. Dynamic content (files, retrieved memory, skills) enters the Prompt through a [PromptLoading](../patterns/prompt-loading.md) strategy. The Prompt is the common denominator — the shape everything converges into before the LLM runs.

Treating Prompt as a primitive (rather than a composition) is deliberate: the protocol structure is not defined by other theory concepts. It's defined by the LLM's training. Our concepts _fill_ Prompt slots; they don't _compose_ into Prompt. Two agents using different provider protocols have different Prompt shapes even if they use identical Sessions and Tools.

## Formal Definition

```typescript
// Conceptual shape — protocol-specific serialization produces the actual string
type Prompt = {
  system?: string; // agent identity, instructions, always-loaded context
  tools?: ToolSchema[]; // callable tools the model should know about
  messages: Message[]; // per-call snapshot of Session.history plus any pending user input
  inserts?: string[]; // dynamic content injected per-turn as raw text (retrieved docs, tool results)
};

type Message =
  | { role: "user"; content: string }
  | { role: "assistant"; content: string; toolCalls?: ToolCall[] }
  | { role: "tool_result"; callId: string; result: string };

type ToolCall = { id: string; name: string; args: Record<string, unknown> };

type ToolSchema = {
  name: string;
  description: string; // the LLM reads this to decide if/when to call
  input: JSONSchema; // JSON Schema — standard format; not a theory concept
};

// Serialization — protocol-specific, defined by the LLM provider
type Protocol = "anthropic" | "openai" | "xml" | "raw-text";

const serialize = (prompt: Prompt, protocol: Protocol): string => {
  /* ... */
};
```

The actual input to the LLM is always a string (or token sequence). The Prompt's structured form is what agent code manipulates; serialization to the final string is the protocol's job. Provider SDKs usually hide the serialization — you pass structured messages and tools, and the SDK formats them into whatever the model expects.

---

## Prompt vs. LLM

Two things are easily conflated — keep them distinct:

| Concept    | What it is                                                                                     |
| ---------- | ---------------------------------------------------------------------------------------------- |
| **LLM**    | A stateless function: takes a Prompt, produces a stream of output text.                        |
| **Prompt** | The structured input: system + tools + history + inserts, serialized to a string per protocol. |

The LLM itself is dumb about structure. Everything structured lives in the Prompt (input) or gets parsed out of the output text stream (on the way back). The output side is not a named concept — it's just text that the [AgentLoop](../harness/agent-loop.md) reads, splitting final content from tool-call-shaped blocks per the same protocol the Prompt used on the way in.

---

## What Goes In a Prompt

A Prompt is assembled each time the LLM is called. The pieces come from different sources:

| Piece                | Source                                                                                                                                    |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| System instructions  | Agent configuration — often composed from multiple sources (CLAUDE.md, agent role, phase instructions)                                    |
| Tool schemas         | The set of Tools the Agent currently exposes                                                                                              |
| Messages (this turn) | Snapshot of [Session](../harness/session.md).history plus any pending user input                                                          |
| Dynamic inserts      | Per-turn content injected by a [PromptLoading](../patterns/prompt-loading.md) strategy (retrieved memory, skill expansions, tool outputs) |

Who fills each slot is a design choice — that's the space [PromptLoading](../patterns/prompt-loading.md) covers. Eager inserts (CLAUDE.md, project structure) go into `system`. Lazy inserts (skills) load on-demand into `messages` or `system`. Per-turn dynamic inserts (retrieved memory, tool results) arrive fresh each call.

---

## Protocol as the Contract Between LLM and Prompt

Different LLM providers implement different Prompt protocols:

- **Anthropic**: `system`, `messages[]`, `tools[]` at the API level; tool-call output comes back as `tool_use` blocks.
- **OpenAI**: `messages[]` with roles (`system`, `user`, `assistant`, `tool`), `tools[]` for function calling; tool-call output in `tool_calls` field.
- **Raw text / legacy**: some older setups format everything into a single string with special tokens (`<|im_start|>`, etc.).
- **XML-prompted models**: some models work best with explicit XML tags (`<task>`, `<tool_call>`) in a raw string.

The LLM's training determines which protocol it speaks. The Prompt must serialize accordingly. Running the same Session against two different LLMs may require different Prompt serialization — the Prompt is where that difference lives.

---

## Prompt and Tool Definitions

A Tool schema in the Prompt is not the Tool itself. The Tool is a function in agent-side code (`execute(params) → result`). What the Prompt carries is the Tool's **schema** — name, description, input shape — so the LLM knows the Tool exists and how to invoke it. When the LLM's output contains a tool call, agent-side code matches the name to the actual Tool and runs its `execute`. The Prompt says "here's what's available"; the AgentLoop says "let's run what the LLM chose."

This separation is why bash is such a distinctive Tool: its schema in the Prompt is a single input (`command: string`), but its actual capability is whatever the [Environment](../environment/environment.md) allows. The Prompt's tool schema is narrow; the Tool's real power is wide.

---

## Related

- [LLM](llm.md) — the function the Prompt feeds into
- [Tool](tool.md) — what Tool schemas in the Prompt reference
- [Session](../harness/session.md) — source of the message history in the Prompt
- [PromptLoading](../patterns/prompt-loading.md) — strategies that fill Prompt slots (eager, lazy, progressive, per-turn)
- [AgentLoop](../harness/agent-loop.md) — assembles the Prompt each iteration, parses the LLM's output stream
