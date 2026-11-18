<a id="verification-strategies"></a>

# Verification methods

A verification method is one type of test for an agent system. The
[seams](seams.md) tell where a test can replace a part. A method tells the
checks that the test does and when the test runs. Each method uses the trace,
the seams and the checks differently. Thus each method finds a type of error
that the other methods do not find.

[The verification stack](index.md#the-verification-stack) compares the methods.

<a id="strategy-shell-testing"></a>

## Shell tests

A shell test replaces the LLM with a mock. Then it examines all the
deterministic code of the system, from the input to the output.

- **Checks**: all the deterministic steps. Examples are the assembly of the
  prompt, the changes to the data and the input and output of the channels.
  Other examples are the format of the output, the file writes and the changes
  of phase.
- **When**: at each change of the code. Shell tests are fast, free and
  deterministic.

```typescript
const mockLlm = createMock({ responses: ["# Mock Report\n\nNarrative."] });
const workspace = createTempWorkspace(fixtureData);

await reportPipeline.run(workspace, { llm: mockLlm });

// Deterministic checks:
assert(mockLlm.calls.length === 1); // LLM called once
assert(mockLlm.calls[0].stdinData.includes("wallet data")); // correct input assembled
assert(workspace.read("report.md") === "# Mock Report\n\nNarrative."); // output written
assert(workspace.exists("dimensions/wallet-summaries.json")); // intermediate files written
```

A shell test examines all the system, but not the operation of the LLM. It finds
these bugs:

- code that changes the data incorrectly
- files that are not there
- an incorrect format of the prompt
- tools with incorrect connections
- incorrect changes of phase.

These bugs go to the users most frequently. They are errors in the connections
between the parts, not errors of reasoning.

**Limit**: a shell test does not tell if the LLM gives a good output for the
assembled prompt.

<a id="strategy-property-based-output-verification"></a>

## Property checks

A property check runs the LLM and examines the structural properties of the
output, not its words.

- **Checks**: the output has the necessary sections. It parses as correct JSON
  or Markdown. It includes each item of the input. Its length is in the limits.
- **When**: at each change of the prompt or of the model version, or at regular
  intervals. Property checks are slow and not deterministic. The model provider
  charges for each call.

```typescript
const result = await reportPipeline.run(realWorkspace, { llm: realLlm });
const report = workspace.read("report.md");

// Property checks, not text comparison:
assert(report.length > 500); // not empty or cut off
assert(report.includes("## Summary")); // has the necessary section
assert(report.includes("Test Wallet")); // names the wallet
assert(!report.includes("undefined")); // no template leftovers
assert(!report.includes("0xaaaa1111")); // addresses replaced with labels

// For structured output:
const parsed = JSON.parse(workspace.read("analysis.json"));
assert(parsed.wallets.length === expectedWalletCount);
assert(parsed.wallets.every((w) => w.totalUsd >= 0)); // no negative balances
```

Property checks accept outputs that are not deterministic. The LLM can use
different words, if the structural properties stay correct. Property checks find
these errors:

- A change of the prompt removes a section from the output.
- A new model version gives worse outputs.
- Some input data breaks the output, for example an unusual input.

**Rule**: examine the facts that must be correct, not the text of the output.
"Report mentions all wallets" is a property. "Report starts with 'This portfolio
consists of'" is not a property. It compares the text, and a small change of the
words breaks it.

<a id="strategy-llm-as-judge"></a>

## LLM judge

An LLM judge is a second LLM that examines the output of the first LLM against a
list of rules. It is the [Evaluator](../patterns/evaluator.md) pattern, but it
runs in the tests, not in the run of the agent.

- **Checks**: the quality properties that structural checks cannot examine. For
  example, the text is clear, the facts are correct, the text includes all the
  input, and the words are applicable to the reader.
- **When**: at regular intervals, or at each change of the prompt or of the
  model. Each test uses two LLM calls. Thus the model provider charges the most
  for this method.

```typescript
type EvalCriterion = {
  name: string;
  description: string; // what a good output looks like
  scoringGuide: string; // scale from 1 to 5
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

One [Evaluator](../patterns/evaluator.md) implementation can do two tasks. It
corrects the agent at runtime, and it monitors the quality in the tests. Write
the rules one time, and use them in the two modes:

```
Runtime Evaluator:   trigger=auto    → output=feedback loop (retry)
Offline Evaluator:   trigger=CI/cron → output=score tracking (alert on regression)
```

<a id="strategy-trajectory-verification"></a>

## Trajectory checks

A trajectory check examines the sequence of the steps of the agent, not only the
last output.

- **Checks**: the agent selects the correct tools in the correct sequence. It
  does not make calls that have no value. It stays in the limits of LLM calls
  and tokens.
- **When**: at each change to the instructions of the agent, for example the
  system prompt, the tool definitions or the Guardrails. A trajectory check must
  have the [trace](trace.md) of the run.

```typescript
const trace = await runWithTracing(agent, task);

// Tool selection:
assert(trace.steps.some((s) => s.toolResults.find((t) => t.tool === "grep")));

// Sequence:
const toolSequence = trace.steps.flatMap((s) =>
  s.toolResults.map((t) => t.tool),
);
const grepIndex = toolSequence.indexOf("grep");
const writeIndex = toolSequence.indexOf("write_file");
assert(grepIndex < writeIndex); // searched before writing

// Cost limits:
assert(trace.metadata.totalLlmCalls <= 10); // no endless loop
assert(trace.metadata.totalTokens < 50_000); // context does not grow out of control

// No prohibited operations:
assert(
  trace.steps.every(
    (s) =>
      !s.toolResults.find(
        (t) => t.tool === "bash" && t.params.command.includes("rm -rf"),
      ),
  ),
);
```

Trajectory checks find bugs that a check of only the output cannot find:

- The agent gives the correct result, but it uses 50 LLM calls for it. The model
  provider charges for each of these calls.
- The agent reads the correct file, but it also reads 30 files that are not
  related to the task. It does much work that has no value.
- The agent removes a file and writes all of it again. It does not change only
  the necessary part. This procedure breaks easily.

**Relation to [Guardrails](../harness/guardrail.md)**: Guardrails stop bad
operations during the run. Trajectory checks make sure after the run that the
operations were correct. The two layers operate together. Guardrails are limits
at runtime. Trajectory checks examine the same operations after the run.

<a id="strategy-snapshot--golden-file-testing"></a>

## Snapshot tests

A snapshot test keeps a known correct output or trace as a reference: a golden
file. The test fails if a new run is too different from the golden file.

- **Checks**: the system gives the same outputs, or almost the same outputs, at
  each run.
- **When**: at each change of the code, for deterministic snapshots, for example
  of the prompt. For snapshots of LLM output, run the test at regular intervals.
  These tests compare by meaning, not by text, and accept small differences.

```typescript
// Snapshot the full prompt sent to the LLM (deterministic)
const trace = await runWithTracing(agent, task, { llm: mockLlm });
const prompt = trace.steps[0].llmCall.messagesIn;
expect(prompt).toMatchSnapshot("wallet-report-prompt");
// Fails if the prompt changes by accident

// Snapshot LLM output with a comparison by meaning (non-deterministic)
const output = await runWithRealLlm(agent, task);
const similarity = cosineSimilarity(embed(output), embed(goldenOutput));
assert(similarity > 0.85); // close enough to the golden file
```

Snapshots of the prompt are free, and they give more value than each other
verification method. The prompt is the most important item of an agent system,
because it controls all the work of the LLM. A snapshot test makes sure that no
change of the code changes the prompt by accident. Examples are a renamed
variable in the text of the prompt, a removed context provider and a change of
the format. These tests are also deterministic.

## Related

- [Verification](index.md): the parts of verification and how they connect.
- [The trace](trace.md): the data that these methods examine.
- [Seams](seams.md): the points where these methods replace a part.
- [Evaluator](../patterns/evaluator.md): the pattern that the LLM judge uses
  again.
- [Guardrail](../harness/guardrail.md): the runtime layer that operates together
  with trajectory checks.
