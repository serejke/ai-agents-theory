# AgentLoop

AgentLoop is the recursive cycle that connects [LLM](../primitives/llm.md) reasoning to [Tool](../primitives/tool.md) execution. It is the harness that turns the two primitives into a running agent — the minimum machinery required for autonomous behavior.

**Composition**: LLM + Tool[] in a recursive cycle.

## Why It Matters

Neither the LLM nor Tools alone produce agent behavior. The LLM can reason about what to do but cannot act. Tools can act but cannot decide what to do. AgentLoop closes the gap: the LLM decides which tool to call, the tool executes and returns a result, the LLM sees the result and decides what to do next. This cycle continues until the LLM produces a final text response with no tool calls.

AgentLoop is harness, not a compositional choice. Every "AI assistant" — Copilot, ChatGPT, Claude.ai, any product that lets you chat with an AI that can take actions — runs on some instance of it. Systems differ in which tools and which system prompt they wire in; the loop itself is structural.

## Formal Definition

```typescript
// Events emitted during a run: text chunks as the LLM produces them,
// tool calls detected in the output stream, tool results after execution.
type Event =
  | { kind: "text"; delta: string }
  | { kind: "tool_call"; call: ToolCall }
  | { kind: "tool_result"; callId: string; result: ToolResult };

type AgentLoop = (
  llm: LLM,
  tools: Tool[],
  system: string,
) => (messages: Message[]) => AsyncGenerator<Event>;

// Semantics:
// 1. Assemble a Prompt from (system, messages, tool schemas)
// 2. Call llm(prompt); stream text; parse tool calls from the output
// 3. If no tool calls → return
// 4. Execute each tool; append results to messages; goto 1
```

The loop is a while loop, not a graph, not a pipeline, not a state machine. It's the simplest possible control flow that produces autonomous behavior:

```typescript
// Implementation sketch (~20 lines)
async function* agentLoop(llm, tools, system, messages) {
  while (true) {
    const prompt = {
      system,
      tools: tools.map((t) => t.schema),
      messages,
    };
    // llm(prompt) streams text; the parser splits it into content + tool calls
    // per the active protocol.
    const { text, toolCalls } = await runAndParse(llm, prompt);
    if (toolCalls.length === 0) {
      yield { kind: "text", delta: text };
      return;
    }
    messages.push({ role: "assistant", content: text, toolCalls });
    for (const call of toolCalls) {
      const tool = tools.find((t) => t.name === call.name);
      const result = await tool.execute(call.args);
      messages.push({ role: "tool_result", callId: call.id, result });
    }
  }
}
```

Some frameworks model this loop as a two-node graph (call_model ↔ call_tools). The graph representation adds compile steps, node definitions, and edge routing for something that is fundamentally "call LLM, if tools then execute and repeat." The graph model pays off for complex multi-phase workflows (see [StateMachine](../patterns/state-machine.md)), but for the base tool-use loop, a while loop is the right abstraction.

---

## The Minimal Agent

AgentLoop is the **minimal agent**. Give it an LLM and `[fileRead, fileWrite, bash]` and you have a coding assistant. Give it `[webSearch, webFetch]` and you have a research assistant. The architecture is identical — the capability surface is entirely determined by the tool set and the system prompt.

```typescript
// A coding agent
const coder = agentLoop(
  claude_sonnet,
  [fileRead, fileWrite, bash, grep, glob],
  codingPrompt,
);

// A research agent — same loop, different tools and prompt
const researcher = agentLoop(
  claude_sonnet,
  [webSearch, webFetch, noteWrite],
  researchPrompt,
);
```

This is why understanding AgentLoop matters: once you see that the loop is universal, you stop thinking about "coding agents" and "research agents" as different architectures. They're the same loop with different configurations.

---

## What AgentLoop Lacks

AgentLoop is stateless across invocations — it takes `messages` as input and produces events as output, but it doesn't maintain conversation history between calls. Each invocation is independent.

This is by design: AgentLoop is a pure computation. To get a conversational agent that remembers what you said earlier in the chat, you need [Session](session.md) — which wraps AgentLoop with a persistent `history: Message[]` accumulator.

To get an agent that remembers across conversations, you add [Memory](../patterns/memory.md). To get an agent with hard safety constraints, you wrap its tools with [Guardrails](../harness/guardrail.md). To get a multi-phase workflow, you chain multiple AgentLoops via [StateMachine](../patterns/state-machine.md). AgentLoop is the foundation — everything else composes on top of it.

---

## Related

- [LLM](../primitives/llm.md) — the reasoning component of the loop
- [Tool](../primitives/tool.md) — the action component of the loop
- [Session](session.md) — AgentLoop + history = a conversational agent
- [Guardrail](../harness/guardrail.md) — constraints that wrap tool execution within the loop
