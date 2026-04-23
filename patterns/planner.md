# Planner

A Planner is a [Session](../harness/session.md) configured with read-only tools and a system prompt that requires structured output — a plan, not an implementation. It is the pattern behind every system where the agent first thinks, then acts.

**Composition**: Session + constraint (read-only tools + structured output prompt).

## Why It Matters

Letting an agent immediately start writing code or making changes is risky for non-trivial tasks. The agent may misunderstand the problem, miss important context, or take an approach that contradicts existing architecture. A Planner forces the agent to explore the codebase and propose a structured plan _before_ any writes happen. The plan can be reviewed by a human, by an [Evaluator](evaluator.md), or by a [StateMachine](state-machine.md) guard before execution proceeds.

## Formal Definition

```typescript
// Planner is an AgentLoop with a specific configuration:
// read-only tools + system prompt requiring structured output
const planner = createSession({
  llm: claude_sonnet,
  tools: [fileRead, grep, glob], // read-only! Doesn't write code.
  system: `You are an architect. Given a task:
    1. Explore the codebase
    2. Propose an implementation plan as a list of steps
    3. For each step, specify files and what to change
    DO NOT write code. Plan only.`,
});

// Usage:
const plan = await planner.send("reuse timeline on details page");
// plan = structured text with steps
```

Planner is a Session with a specific configuration (read-only tools + system prompt requiring structured output). Claude Code plan mode, Codex reasoning — all implement this pattern.

---

## Planner in Context

Planner typically appears as a phase in a [StateMachine](state-machine.md):

```typescript
{
  phases: {
    plan: { tools: readOnlyTools, system: planningPrompt },   // Planner
    approve: { trigger: "human_approval" },                   // gate
    implement: { tools: allTools, system: implementPrompt },  // full access
  }
}
```

The value of Planner is not just in the plan itself, but in the separation of concerns it enables: the planning phase uses cheap, fast, read-only operations to explore the problem space; the implementation phase uses expensive, slower, write-capable tools to execute. This separation maps naturally to model routing — planning might use a cheaper model, implementation a more capable one (see [Router](router.md)).

---

## Related

- [Session](../harness/session.md) — the harness layer that Planner constrains
- [StateMachine](state-machine.md) — Planner is typically one phase in a multi-phase workflow
- [Router](router.md) — may select different models for planning vs. implementation
- [Evaluator](evaluator.md) — may assess the quality of a plan before approving execution
