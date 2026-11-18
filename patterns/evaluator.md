# Evaluator

An Evaluator examines the output of an agent against a list of conditions and
gives a verdict. Usually an LLM examines the output, but a different mechanism
can do it too, and the agent can use the verdict to try again. This pattern lets
an agent correct its work: the agent does not stop after its first result, but
continues until the result is good.

**Composition**: LLM + loop (a function that examines the result + a feedback
cycle).

<a id="why-it-matters"></a>

## Value

Without an Evaluator, the first result of an agent is its last result. With an
Evaluator, the agent can do the work again and again:

1. It makes a result.
2. The Evaluator examines the result and finds the problems.
3. The agent corrects the problems.
4. The Evaluator examines the result again.

This sequence is a **feedback loop**: a closed cycle that changes the operation
of the system very much. Each system that does the work again until the result
is good uses this pattern.

<a id="formal-definition"></a>

## Definition

```typescript
type Evaluator = (
  task: string, // the result that the user wants
  result: string, // the result that the agent made
  evidence?: Evidence, // screenshot, tests, diff
) => Promise<Verdict>;

type Verdict = {
  pass: boolean;
  issues: string[]; // the problems
  suggestions: string[]; // how to correct them
};
```

<a id="simple-evaluator--one-llm-call"></a>

### Evaluator with one LLM call

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

<a id="visual-evaluator--multimodal"></a>

### Evaluator that examines an image

The LLM of this Evaluator reads text and images. It compares the task with an
image of the result.

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

## The feedback loop

An Evaluator gives its full value only when its verdict goes back to the agent.
This closes the loop:

```typescript
// Without an Evaluator: one pass
const result = await coder.send(task);

// With an Evaluator: more passes until the result is good
let result = await coder.send(task);
for (let i = 0; i < maxRetries; i++) {
  const verdict = await evaluator(task, result, await getScreenshot());
  if (verdict.pass) break;
  result = await coder.send(`Fix: ${verdict.issues.join(", ")}`);
}
```

The structure of the Evaluator is a composition of an LLM and a loop, thus the
Evaluator is a pattern. But its function is new: the agent corrects its work.
This function does not occur in the LLM or in the loop if they operate
independently. It occurs only when they operate together.

<a id="evaluator-as-statemachine-trigger"></a>

## The Evaluator in a StateMachine

Frequently, the verdict of an Evaluator causes the change from one phase to the
next phase in a [StateMachine](state-machine.md):

```typescript
{
  from: "verify",
  to: "deliver",
  trigger: "evaluation_passed"
}
{
  from: "verify",
  to: "implement",
  trigger: "evaluation_failed"  // go back and correct the work
}
```

The verdict of the Evaluator selects the next phase: the workflow goes forward,
or it goes back for one more cycle. Thus, the change of phase is the same
feedback loop.

## Related

- [LLM](../primitives/llm.md): the part that examines the output when an LLM is
  the Evaluator
- [AgentLoop](../harness/agent-loop.md): the loop that the Evaluator puts in a
  cycle of retries
- [StateMachine](state-machine.md): the verdicts of an Evaluator frequently
  cause a change of phase
- [Verification](../verification/index.md): uses the Evaluator pattern after the
  run, to monitor the quality of an agent
