# Evaluator

An Evaluator uses an LLM (or other assessment mechanism) to judge an agent's output against criteria, optionally feeding the verdict back into a retry loop. It is the pattern that enables self-correction — the qualitative difference between "single pass" and "iterate until good enough."

**Composition**: LLM + loop (assessment function + feedback cycle).

## Why It Matters

Without an Evaluator, an agent's first attempt is its final answer. With an Evaluator, the agent can iterate: produce a result, assess it, identify issues, fix them, reassess. This creates a **feedback loop** — a closed cycle that qualitatively changes system behavior. All systems that "iterate until result" use this pattern.

## Formal Definition

```typescript
type Evaluator = (
  task: string, // what was wanted
  result: string, // what was produced
  evidence?: Evidence, // screenshot, tests, diff
) => Promise<Verdict>;

type Verdict = {
  pass: boolean;
  issues: string[]; // what's wrong
  suggestions: string[]; // how to fix
};
```

### Simple Evaluator — one LLM call

```typescript
const codeEvaluator: Evaluator = async (task, result) => {
  return await llm("evaluate", [
    {
      role: "user",
      content: `Task: ${task}\nResult: ${result}\nDoes it match?`,
    },
  ]);
};
```

### Visual Evaluator — multimodal

```typescript
const visualEvaluator: Evaluator = async (task, result, evidence) => {
  return await llm("evaluate", [
    {
      role: "user",
      content: [
        { type: "text", text: `Task: ${task}` },
        { type: "image", data: evidence.screenshot },
      ],
    },
  ]);
};
```

---

## The Feedback Loop

The Evaluator's power comes from closing the loop:

```typescript
// Without Evaluator: single pass
const result = await coder.send(task);

// With Evaluator: iterations until convergence
let result = await coder.send(task);
for (let i = 0; i < maxRetries; i++) {
  const verdict = await evaluator(task, result, await getScreenshot());
  if (verdict.pass) break;
  result = await coder.send(`Fix: ${verdict.issues.join(", ")}`);
}
```

Formally, the Evaluator is a composition of LLM + loop. But it produces **emergent behavior** (self-correction) that doesn't exist in any of the base primitives individually. This puts it on the edge between pattern and primitive — it's structurally a composition, but functionally it introduces a qualitatively new capability.

---

## Two Deployment Modes

The same Evaluator implementation serves two different purposes depending on when and how it runs:

```
Runtime Evaluator:   trigger = auto       → output = feedback loop (retry on failure)
Offline Evaluator:   trigger = CI / cron  → output = score tracking (alert on regression)
```

**Runtime**: The Evaluator runs as part of the agent's execution. If the verdict fails, the agent retries. This is self-correction — the Evaluator is in the loop.

**Offline**: The Evaluator runs after the fact, as part of a [Verification](../verification.md) suite. It scores the output against criteria and tracks quality over time. If scores drop below a threshold, it alerts — but doesn't retry. This is quality monitoring.

The pattern is identical. The deployment is different. This means eval infrastructure and runtime evaluation can share the same Evaluator implementation — define criteria once, use them for both self-correction and offline quality tracking.

```typescript
// Define once:
const criteria: EvalCriterion[] = [
  {
    name: "accuracy",
    description: "All facts consistent with input data",
    scoringGuide: "5=perfect, 3=minor inaccuracies, 1=hallucinated",
  },
  {
    name: "completeness",
    description: "Covers all wallets and significant transactions",
    scoringGuide: "5=all covered, 3=some missing, 1=most missing",
  },
];

// Use for runtime self-correction:
const runtimeEval = createEvaluator(criteria, { onFail: "retry" });

// Use for offline quality tracking:
const offlineEval = createEvaluator(criteria, { onFail: "alert" });
```

---

## Evaluator as StateMachine Trigger

Evaluators frequently serve as transition triggers in a [StateMachine](../primitives/state-machine.md):

```typescript
{
  from: "verify",
  to: "deliver",
  trigger: "evaluation_passed"
}
{
  from: "verify",
  to: "implement",
  trigger: "evaluation_failed"  // loop back
}
```

The Evaluator determines whether the workflow advances or loops back for another iteration. This is the self-correction feedback loop expressed as a state transition.

---

## Related

- [LLM](../primitives/llm.md) — the assessment engine in an LLM-as-judge evaluator
- [AgentLoop](agent-loop.md) — what the Evaluator wraps with a retry loop
- [StateMachine](../primitives/state-machine.md) — Evaluator verdicts often trigger state transitions
- [Verification](../verification.md) — offline deployment of the Evaluator pattern for quality tracking
