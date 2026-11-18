# Planner

A Planner is an agent that examines a task and writes a plan, but does not
change the code or the files. It is a [Session](../harness/session.md) with
tools that only read and a system prompt that tells the LLM to write a plan in a
specified structure. Each system in which the agent first thinks and then does
the work uses this pattern.

**Composition**: Session + limits (read-only tools + a system prompt that tells
the LLM to write a plan in a specified structure).

<a id="why-it-matters"></a>

## Value

A Planner lets a person or a system examine the method of an agent before the
agent changes a file. If a task is not small, an agent that starts to write code
immediately can cause problems:

- It can read the problem incorrectly.
- It can ignore important context.
- It can select a method that does not agree with the structure of the code.

A Planner makes the agent examine the code and write a plan in a specified
structure before it writes a file. Then a person, an [Evaluator](evaluator.md)
or a rule of a [StateMachine](state-machine.md) can examine the plan before the
work continues.

<a id="formal-definition"></a>

## Definition

```typescript
// A Planner is a Session with read-only tools
// and a system prompt that asks for a plan in a fixed structure.
const planner = createSession({
  llm: claude_sonnet,
  tools: [fileRead, grep, glob], // read-only: it does not write code
  system: `You are an architect. Given a task:
    1. Explore the codebase
    2. Propose an implementation plan as a list of steps
    3. For each step, specify files and what to change
    DO NOT write code. Plan only.`,
});

// Use:
const plan = await planner.send("reuse timeline on details page");
// plan = text with steps in a fixed structure
```

Examples of this pattern are the plan mode of Claude Code and the reasoning step
of Codex.

<a id="planner-in-context"></a>

## The Planner as a phase

Usually, the Planner is one phase of a [StateMachine](state-machine.md):

```typescript
{
  phases: {
    plan: { tools: readOnlyTools, system: planningPrompt },   // Planner
    approve: { trigger: "human_approval" },                   // a person approves the plan
    implement: { tools: allTools, system: implementPrompt },  // full access
  }
}
```

The plan is not the only value of a Planner. The Planner also divides the work
into two phases with different tools:

- The plan phase examines the problem with read-only operations. These are fast
  operations with a low cost.
- The implementation phase does the work with tools that can write. These tools
  are slower and their cost is higher.

Thus, the two phases can use different models. The plan phase can use a model
with a lower cost, and the implementation phase can use a stronger model. A
[Router](router.md) can select the model for each phase.

## Related

- [Session](../harness/session.md): the harness that the Planner puts limits on
- [StateMachine](state-machine.md): the Planner is usually one phase of a
  workflow with two or more phases
- [Router](router.md): can select different models for the plan and for the
  implementation
- [Evaluator](evaluator.md): can examine the quality of a plan before the work
  starts
