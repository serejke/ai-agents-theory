# Verification

Verification is how you confirm that an agent system works. It is an orthogonal concern — not a primitive or pattern, but a set of strategies that intersect every primitive at its boundaries.

## Why It Matters

Traditional software is deterministic. Given the same input, the same code produces the same output. Testing is straightforward: call a function, assert on the result. If the test passes today, it passes tomorrow.

Agent systems break this contract. The core computation — the [LLM](primitives/llm.md) — is non-deterministic. The same prompt may produce different reasoning, different tool call sequences, different outputs. The [AgentLoop](patterns/agent-loop.md) compounds this: each tool call produces a result that the LLM interprets to decide the next action, so small variations early in a run cascade into divergent trajectories.

The answer is not "you can't test agents." The answer is that agent systems have multiple layers, most of which are deterministic, and the non-deterministic core requires different verification strategies than the deterministic shell. Understanding which layer you're verifying — and what tools apply at each — is the difference between a system you trust and one you hope works.

---

## Trace — The Observable Output of AgentLoop

Before you can verify an agent, you need to observe what it did. The final output (a generated report, a code change, a message) is only part of the story. Two runs may produce identical outputs via completely different paths — one by reasoning correctly, the other by getting lucky.

The full observable behavior of an AgentLoop is its **trace**: the ordered sequence of LLM calls, tool invocations, and intermediate results.

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

Trace is to AgentLoop what a stack trace is to a function call — it tells you not just what happened, but _how_ and _why_. Every verification strategy operates on some subset of the trace:

| Verification type       | What it inspects                                     |
| ----------------------- | ---------------------------------------------------- |
| Output verification     | `trace.finalOutput` only                             |
| Trajectory verification | `trace.steps` — the sequence of actions              |
| Cost verification       | `trace.metadata` — tokens, calls, duration           |
| Prompt verification     | `trace.steps[0].llmCall.messagesIn` — what was asked |

A system that doesn't capture traces can only do output verification. One that does can verify everything.

---

## Seams — Where Primitives Meet Is Where You Test

Every agent system is a composition of primitives. At each boundary between primitives, there's a **seam** — a point where you can substitute one implementation for another without changing the rest of the system.

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

The compositional nature of agent systems — the fact that they're assembled from discrete, substitutable primitives — is what makes them testable at all. A monolithic agent with no seams would be a black box. An agent built from primitives is a composition of individually verifiable components with a non-deterministic core that requires statistical verification.

### The LLM Seam

The most important seam in any agent system is between the LLM and everything else. The LLM is the only non-deterministic primitive. Everything else — tool execution, channel reads/writes, guardrail checks, state machine transitions — is deterministic.

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

### The Channel Seam

In multi-agent pipelines, the [Channel](primitives/channel.md) between agents is a natural seam. Each channel write produces an artifact; each channel read consumes one. You can verify each agent in isolation by pre-populating its input channel and asserting on its output channel.

```typescript
// Test Agent B in isolation:
const inputChannel = createChannel();
inputChannel.write("notes/topic-1.md", fixtureNote); // pre-populated

const outputChannel = createChannel();
await agentB.run({ input: inputChannel, output: outputChannel });

assert(outputChannel.read("analysis.md").includes("key finding"));
```

This works for any channel variant — filesystem, shared state, artifact store.

### The Tool Boundary Seam

[Tools](primitives/tool.md) are side effects. In production, a tool might call an external API, write to a database, or execute a shell command. In tests, tools are the outermost seam — you can replace them with fakes that record what was called and return canned results.

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

## Verification Strategies

The seams above define _where_ to test. The strategies below define _what_ to assert and _when_ to run it.

### Strategy 1: Shell Testing

**What:** Replace the LLM with a mock. Test all deterministic logic end-to-end.

**Asserts on:** Prompt assembly, data transforms, channel I/O, output formatting, file writes, state transitions.

**When:** Every code change. Fast, free, deterministic.

```typescript
const mockLlm = createMock({ responses: ["# Mock Report\n\nNarrative."] });
const workspace = createTempWorkspace(fixtureData);

await reportPipeline.run(workspace, { llm: mockLlm });

// Deterministic assertions:
assert(mockLlm.calls.length === 1); // LLM called once
assert(mockLlm.calls[0].stdinData.includes("wallet data")); // correct input assembled
assert(workspace.read("report.md") === "# Mock Report\n\nNarrative."); // output written
assert(workspace.exists("dimensions/wallet-summaries.json")); // intermediaries produced
```

This tests everything _except_ the LLM's behavior. It catches: broken data transforms, missing files, incorrect prompt templates, wrong tool wiring, broken state transitions. These are the bugs that ship most often — plumbing failures, not reasoning failures.

**Limitation:** Says nothing about whether the real LLM would produce good output for the assembled prompt.

### Strategy 2: Property-Based Output Verification

**What:** Run the real LLM. Assert on _structural properties_ of the output, not exact text.

**Asserts on:** Output contains required sections. Output parses as valid JSON/markdown. Output mentions all input entities. Output stays within length bounds.

**When:** On prompt changes, model upgrades, or scheduled (nightly). Slow, costs money, non-deterministic.

```typescript
const result = await reportPipeline.run(realWorkspace, { llm: realLlm });
const report = workspace.read("report.md");

// Property assertions (not exact text):
assert(report.length > 500); // not empty/truncated
assert(report.includes("## Summary")); // has required section
assert(report.includes("Test Wallet")); // mentions the wallet
assert(!report.includes("undefined")); // no template artifacts
assert(!report.includes("0xaaaa1111")); // addresses resolved to labels

// For structured output:
const parsed = JSON.parse(workspace.read("analysis.json"));
assert(parsed.wallets.length === expectedWalletCount);
assert(parsed.wallets.every((w) => w.totalUsd >= 0)); // no negative balances
```

Property assertions tolerate non-determinism — the LLM can rephrase freely as long as structural properties hold. This catches: prompt regressions (changed prompt → missing section), model degradation (new model version → worse outputs), data-dependent failures (edge case input → broken output).

**Key principle:** Assert on what _must_ be true, not what the output _looks like_. "Report mentions all wallets" is a property. "Report starts with 'This portfolio consists of'" is a brittle match.

### Strategy 3: LLM-as-Judge

**What:** Use a second LLM to evaluate the first LLM's output against criteria. This is the [Evaluator](patterns/evaluator.md) pattern deployed offline.

**Asserts on:** Quality dimensions — coherence, accuracy, completeness, tone — that can't be captured with structural assertions.

**When:** Scheduled, or on prompt/model changes. Expensive (two LLM calls per eval).

```typescript
type EvalCriterion = {
  name: string;
  description: string; // what "good" looks like
  scoringGuide: string; // 1-5 scale rubric
};

const criteria: EvalCriterion[] = [
  {
    name: "accuracy",
    description: "All facts in the report are consistent with the input data",
    scoringGuide: "5=perfect, 3=minor inaccuracies, 1=hallucinated content",
  },
  {
    name: "completeness",
    description: "Report covers all wallets and significant transactions",
    scoringGuide: "5=all covered, 3=some missing, 1=most missing",
  },
];

const judge: Evaluator = async (task, result, evidence) => {
  const verdict = await llm("evaluate", [
    {
      role: "user",
      content: `
      Task: ${task}
      Input data: ${evidence.inputData}
      Agent output: ${result}
      Score each criterion 1-5: ${JSON.stringify(criteria)}
    `,
    },
  ]);
  return parseScores(verdict);
};
```

The same [Evaluator](patterns/evaluator.md) implementation can be used for both runtime self-correction and offline quality tracking — define criteria once, deploy in two modes:

```
Runtime Evaluator:   trigger=auto    → output=feedback loop (retry)
Offline Evaluator:   trigger=CI/cron → output=score tracking (alert on regression)
```

### Strategy 4: Trajectory Verification

**What:** Assert on the _sequence of actions_ the agent took, not just the final output.

**Asserts on:** Correct tool selection, reasonable ordering, no redundant calls, cost bounds.

**When:** On agent logic changes (system prompt, tool definitions, guardrails). Requires trace capture.

```typescript
const trace = await runWithTracing(agent, task);

// Tool selection:
assert(trace.steps.some((s) => s.toolResults.find((t) => t.tool === "grep")));

// Ordering:
const toolSequence = trace.steps.flatMap((s) =>
  s.toolResults.map((t) => t.tool),
);
const grepIndex = toolSequence.indexOf("grep");
const writeIndex = toolSequence.indexOf("write_file");
assert(grepIndex < writeIndex); // searched before writing

// Cost bounds:
assert(trace.metadata.totalLlmCalls <= 10); // not spinning
assert(trace.metadata.totalTokens < 50_000); // not exploding context

// No prohibited actions:
assert(
  trace.steps.every(
    (s) =>
      !s.toolResults.find(
        (t) => t.tool === "bash" && t.params.command.includes("rm -rf"),
      ),
  ),
);
```

Trajectory verification catches a class of bugs invisible to output-only testing: the agent that produces the right answer but takes 50 LLM calls to get there (cost explosion), the agent that reads the right file but also reads 30 irrelevant files (efficiency), the agent that deletes and recreates a file instead of editing it (fragility).

**Relationship to [Guardrails](primitives/guardrail.md):** Guardrails prevent bad actions _during_ execution. Trajectory verification confirms correct behavior _after_ execution. They're complementary: guardrails are runtime constraints, trajectory verification is offline assertions over the same action space.

### Strategy 5: Snapshot / Golden File Testing

**What:** Capture a known-good output (or trace) as a reference. Fail if future runs diverge beyond a threshold.

**Asserts on:** Stability — the system produces similar outputs over time.

**When:** Every code change (for deterministic snapshots like prompts) or periodically (for LLM output snapshots with fuzzy comparison).

```typescript
// Snapshot the exact prompt sent to the LLM (deterministic)
const trace = await runWithTracing(agent, task, { llm: mockLlm });
const prompt = trace.steps[0].llmCall.messagesIn;
expect(prompt).toMatchSnapshot("wallet-report-prompt");
// Fails if prompt changes unexpectedly — catches accidental prompt regressions

// Snapshot LLM output with fuzzy comparison (non-deterministic)
const output = await runWithRealLlm(agent, task);
const similarity = cosineSimilarity(embed(output), embed(goldenOutput));
assert(similarity > 0.85); // similar enough to golden file
```

Prompt snapshots are the highest-value, lowest-cost verification for agent systems. The prompt is the single most important artifact — it determines everything the LLM does. A snapshot test ensures no code change accidentally mutates the prompt (a template variable renamed, a context provider dropped, a formatting change). This is deterministic and free.

---

## The Verification Stack

These strategies compose into a stack, ordered by cost and determinism:

```
                          Cost    Determinism   Frequency
                          ────    ───────────   ─────────
┌─────────────────────┐
│  Shell testing       │  Free    Deterministic  Every commit
│  (mock LLM)          │
├─────────────────────┤
│  Prompt snapshots    │  Free    Deterministic  Every commit
│  (golden files)      │
├─────────────────────┤
│  Property assertions │  $$      Non-det        On prompt/model change
│  (real LLM)          │
├─────────────────────┤
│  Trajectory checks   │  $$      Non-det        On agent logic change
│  (real LLM + trace)  │
├─────────────────────┤
│  LLM-as-judge        │  $$$     Non-det        Scheduled (nightly/weekly)
│  (eval suite)        │
└─────────────────────┘
```

Every layer catches different failures. No single layer is sufficient:

- Shell tests catch plumbing bugs but miss prompt quality issues.
- Property assertions catch structural failures but miss subtle quality regression.
- LLM-as-judge catches quality issues but is expensive and non-deterministic.
- Trajectory checks catch behavioral issues but require trace infrastructure.

A production agent system needs at least the top two layers (shell tests + prompt snapshots). A serious one adds property assertions. A mature one has the full stack.

---

## Design Implications

**For agent builders**: Make the LLM seam explicit from day one. If your agent code calls the LLM directly (hardcoded subprocess call, inline API client), every test requires a real LLM call or monkey-patching. If the LLM is behind a protocol, testing the deterministic shell is trivial. This is the single highest-leverage design decision for testability.

**For pipeline builders**: Capture traces. Even if you don't write trajectory assertions today, having traces means you can debug failures, replay runs, and add assertions later. A trace is cheap to capture and expensive to reconstruct after the fact.

**For eval builders**: Reuse the [Evaluator](patterns/evaluator.md) pattern. If your agent already has runtime self-correction (retry loop with LLM-as-judge), extract the evaluation criteria into a shared definition. Run the same criteria offline for quality tracking. One set of criteria, two deployment modes.

**For the theory**: Verification connects [Guardrails](primitives/guardrail.md) (runtime prevention) with offline quality assurance. Both operate on the same action space — tool calls, outputs, trajectories — but at different points in the lifecycle. A complete agent system has both: guardrails that prevent catastrophic actions during execution, and verification that confirms correct behavior after execution.

---

## Related

- [Guardrail](primitives/guardrail.md) — runtime prevention, complementary to offline verification
- [Evaluator](patterns/evaluator.md) — the pattern reused for LLM-as-judge verification
- [AgentLoop](patterns/agent-loop.md) — what produces the traces that verification inspects
- [Channel](primitives/channel.md) — channel boundaries are natural verification seams
