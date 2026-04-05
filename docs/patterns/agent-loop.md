# AgentLoop

AgentLoop is the recursive cycle that connects [LLM](../primitives/llm.md) reasoning to [Tool](../primitives/tool.md) execution. It is the minimal agent — the simplest composition that produces autonomous behavior.

**Composition**: LLM + Tool[] in a recursive cycle.

## Why It Matters

Neither the LLM nor Tools alone produce agent behavior. The LLM can reason about what to do but cannot act. Tools can act but cannot decide what to do. AgentLoop closes the gap: the LLM decides which tool to call, the tool executes and returns a result, the LLM sees the result and decides what to do next. This cycle continues until the LLM produces a final text response with no tool calls.

Every "AI assistant" — Copilot, ChatGPT, Claude.ai, any product that lets you chat with an AI that can take actions — implements this pattern. The difference between them is in the set of `tools` and contents of `system`, not in the loop itself.

## Formal Definition

```typescript
type AgentLoop = (
  llm: LLM,
  tools: Tool[],
  system: string,
) => (messages: Message[]) => AsyncGenerator<Event>;

// Semantics:
// 1. Call llm(system, messages)
// 2. If response is text only → yield TextDelta events, return
// 3. If response contains toolCalls → execute each tool,
//    append results to messages, goto 1
```

The loop is a while loop, not a graph, not a pipeline, not a state machine. It's the simplest possible control flow that produces autonomous behavior:

```typescript
// Implementation sketch (~20 lines)
async function* agentLoop(llm, tools, system, messages) {
  while (true) {
    const response = await llm(system, messages);
    if (response.toolCalls.length === 0) {
      yield { type: "text", text: response.text };
      return;
    }
    for (const call of response.toolCalls) {
      const tool = tools.find((t) => t.name === call.name);
      const result = await tool.execute(call.params);
      messages.push({ role: "tool_result", callId: call.id, result });
    }
    messages.push({
      role: "assistant",
      content: response.text,
      toolCalls: response.toolCalls,
    });
  }
}
```

Some frameworks model this loop as a two-node graph (call_model ↔ call_tools). The graph representation adds compile steps, node definitions, and edge routing for something that is fundamentally "call LLM, if tools then execute and repeat." The graph model pays off for complex multi-phase workflows (see [StateMachine](../primitives/state-machine.md)), but for the base tool-use loop, a while loop is the right abstraction.

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

This is by design: AgentLoop is a pure computation pattern. To get a conversational agent that remembers what you said earlier in the chat, you need [Session](session.md) — which wraps AgentLoop with a persistent `history: Message[]` accumulator.

To get an agent that remembers across conversations, you need [Memory](../primitives/memory.md). To get an agent with hard safety constraints, you wrap its tools with [Guardrails](../primitives/guardrail.md). To get a multi-phase workflow, you chain multiple AgentLoops via [StateMachine](../primitives/state-machine.md). AgentLoop is the foundation — everything else composes on top of it.

---

## Related

- [LLM](../primitives/llm.md) — the reasoning component of the loop
- [Tool](../primitives/tool.md) — the action component of the loop
- [Session](session.md) — AgentLoop + history = a conversational agent
- [Guardrail](../primitives/guardrail.md) — constraints that wrap tool execution within the loop
