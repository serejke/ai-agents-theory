# Seams

A seam is a primitive boundary where one implementation can be substituted for another without changing the rest of the system. Every agent system is a composition of primitives, and every composition point is a seam.

## Why It Matters

The compositional nature of agent systems — the fact that they're assembled from discrete, substitutable primitives — is what makes them testable at all. A monolithic agent with no seams would be a black box. An agent built from primitives is a composition of individually verifiable components with a non-deterministic core that requires statistical verification.

Every verification [strategy](strategies.md) works by substituting at a seam: replacing the LLM with a mock, pre-populating a channel with fixture data, swapping a tool for a recording fake. Knowing where the seams are tells you where tests can attach.

## The Primitive Seams

```
AgentLoop = LLM + Tool[]
                 ↑ seam: replace LLM with a mock

Session = AgentLoop + history
                       ↑ seam: inject pre-built history

StateMachine = Phase[] + Transition[]
                          ↑ seam: test transitions with synthetic phase outputs

Channel = write → (artifact) → read
                    ↑ seam: pre-populate artifacts, verify what's read

Workspace = filesystem operations
             ↑ seam: use a temporary directory with fixture data
```

## The LLM Seam

The most important seam in any agent system is between the [LLM](../primitives/llm.md) and everything else. The LLM is the only non-deterministic primitive. Everything else — tool execution, channel reads/writes, guardrail checks, state machine transitions — is deterministic.

```typescript
// The LLM seam: a protocol that any implementation can satisfy
type LlmCall = (
  system: string,
  messages: Message[],
) => AsyncStream<TextDelta | ToolCall>;

// Production: real model
const realLlm: LlmCall = (system, messages) =>
  anthropic.stream({ model: "claude-sonnet", system, messages });

// Test: deterministic mock
const mockLlm: LlmCall = (system, messages) =>
  Stream.of({ type: "text", text: "Mock response." });

// Test: recorded replay
const replayLlm: LlmCall = (system, messages) =>
  playbackFromTrace(recordedTrace, messages);
```

Replacing the real LLM with a mock splits the system into two halves:

- **Deterministic shell**: tool execution, data transforms, channel I/O, prompt assembly, output writing, state transitions. Testable with conventional techniques.
- **Non-deterministic core**: LLM reasoning, tool selection, output generation. Requires different strategies (evals, property assertions, trajectory checks).

**You test the shell with mocks, and the core with evals.** The two are complementary, not alternatives.

## The Channel Seam

In multi-agent pipelines, the [Channel](../patterns/channel.md) between agents is a natural seam. Each channel write produces an artifact; each channel read consumes one. You can verify each agent in isolation by pre-populating its input channel and asserting on its output channel.

```typescript
// Test Agent B in isolation:
const inputChannel = createChannel();
inputChannel.write("notes/topic-1.md", fixtureNote); // pre-populated

const outputChannel = createChannel();
await agentB.run({ input: inputChannel, output: outputChannel });

assert(outputChannel.read("analysis.md").includes("key finding"));
```

This works for any channel variant — filesystem, shared state, artifact store.

## The Tool Boundary Seam

[Tools](../primitives/tool.md) are side effects. In production, a tool might call an external API, write to a database, or execute a shell command. In tests, tools are the outermost seam — you can replace them with fakes that record what was called and return canned results.

```typescript
const fakeTools: Tool[] = [
  {
    name: "query_database",
    execute: async (params) => {
      callLog.push({ tool: "query_database", params });
      return { rows: fixtureData };
    },
  },
  {
    name: "write_file",
    execute: async (params) => {
      callLog.push({ tool: "write_file", params });
      return { success: true };
    },
  },
];

// After the agent runs, assert on what tools were called:
assert(callLog.some((c) => c.tool === "query_database"));
assert(
  callLog.find((c) => c.tool === "write_file")?.params.path === "report.md",
);
```

This verifies the agent's _decisions_ — did it call the right tools with the right parameters? — without executing real side effects.

---

## Related

- [Verification](index.md) — the orthogonal concern that uses seams
- [Trace](trace.md) — what substituted components produce for inspection
- [Strategies](strategies.md) — the techniques that exploit seams
- [LLM](../primitives/llm.md) — the primitive behind the most important seam
- [Channel](../patterns/channel.md) — inter-agent seams in multi-agent pipelines
- [Tool](../primitives/tool.md) — the outermost substitution boundary
