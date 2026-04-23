# Trace

A trace is the full observable behavior of an [AgentLoop](../harness/agent-loop.md): the ordered sequence of LLM calls, tool invocations, and intermediate results produced during a run. It is to an AgentLoop what a stack trace is to a function call — it tells you not just what happened, but _how_ and _why_.

## Why It Matters

Before you can verify an agent, you need to observe what it did. The final output (a generated report, a code change, a message) is only part of the story. Two runs may produce identical outputs via completely different paths — one by reasoning correctly, the other by getting lucky. Without a trace, you cannot tell them apart.

A system that doesn't capture traces can only verify outputs. One that does can verify prompts, trajectories, cost, and quality separately.

## Formal Definition

```typescript
type Trace = {
  steps: Step[];
  finalOutput: string;
  metadata: {
    totalLlmCalls: number;
    totalTokens: number;
    durationMs: number;
  };
};

type Step = {
  llmCall: {
    messagesIn: Message[]; // what the LLM saw
    response: string; // what it produced
    toolCalls: ToolCall[]; // what actions it chose
  };
  toolResults: Array<{
    tool: string;
    params: unknown;
    result: ToolResult;
  }>;
};
```

## What Each Verification Type Inspects

Every verification [strategy](strategies.md) operates on some subset of the trace:

| Verification type       | What it inspects                                     |
| ----------------------- | ---------------------------------------------------- |
| Output verification     | `trace.finalOutput` only                             |
| Trajectory verification | `trace.steps` — the sequence of actions              |
| Cost verification       | `trace.metadata` — tokens, calls, duration           |
| Prompt verification     | `trace.steps[0].llmCall.messagesIn` — what was asked |

Traces are cheap to capture at runtime and expensive to reconstruct after the fact. A pipeline that captures traces from day one can add new assertions retroactively; one that doesn't has to re-run every case to verify anything new.

---

## Related

- [Verification](index.md) — the orthogonal concern that consumes traces
- [Seams](seams.md) — the substitution points that produce comparable traces
- [Strategies](strategies.md) — the techniques that assert on traces
- [AgentLoop](../harness/agent-loop.md) — the harness whose execution a trace records
