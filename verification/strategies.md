# Verification Strategies

Distinct techniques for verifying agent systems. [Seams](seams.md) define _where_ to test; strategies define _what_ to assert and _when_ to run it. Each strategy combines trace capture, seam substitution, and assertion differently, and each catches a class of failures the others miss.

## Strategy: Shell Testing

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

## Strategy: Property-Based Output Verification

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

## Strategy: LLM-as-Judge

**What:** Use a second LLM to evaluate the first LLM's output against criteria. This is the [Evaluator](../patterns/evaluator.md) pattern deployed offline.

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

The same [Evaluator](../patterns/evaluator.md) implementation can be used for both runtime self-correction and offline quality tracking — define criteria once, deploy in two modes:

```
Runtime Evaluator:   trigger=auto    → output=feedback loop (retry)
Offline Evaluator:   trigger=CI/cron → output=score tracking (alert on regression)
```

## Strategy: Trajectory Verification

**What:** Assert on the _sequence of actions_ the agent took, not just the final output.

**Asserts on:** Correct tool selection, reasonable ordering, no redundant calls, cost bounds.

**When:** On agent logic changes (system prompt, tool definitions, guardrails). Requires [trace](trace.md) capture.

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

**Relationship to [Guardrails](../harness/guardrail.md):** Guardrails prevent bad actions _during_ execution. Trajectory verification confirms correct behavior _after_ execution. They're complementary: guardrails are runtime constraints, trajectory verification is offline assertions over the same action space.

## Strategy: Snapshot / Golden File Testing

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

## Related

- [Verification](index.md) — the orthogonal concern these strategies implement
- [Trace](trace.md) — the observable surface these strategies assert on
- [Seams](seams.md) — the substitution points these strategies exploit
- [Evaluator](../patterns/evaluator.md) — the pattern reused for LLM-as-judge
- [Guardrail](../harness/guardrail.md) — runtime counterpart to trajectory verification
