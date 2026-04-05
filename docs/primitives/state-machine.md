# StateMachine

A StateMachine defines phases with different capabilities and explicit transition rules between them, including human checkpoints. It is what separates a single continuous conversation from a structured workflow with distinct stages, verification gates, and rollback points.

## Why It Matters

A [Session](../patterns/session.md) is one continuous conversation — the agent has the same tools and the same authority from start to finish. Real workflows need phases: first explore the code (read-only), then get human approval, then implement (full access), then verify (run tests), then deliver (create PR). Each phase has different tools, different permissions, and different success criteria. Transitions between phases may require human approval, evaluation results, or guard conditions.

Without StateMachine, you either build a single all-powerful agent (risky — it can write code before understanding the problem) or manually orchestrate phases by running separate agents and coordinating them yourself. StateMachine is the primitive that makes multi-phase workflows first-class — and arguably the main missing piece for transitioning from "agent in terminal" to "agent as CI/CD pipeline."

## Formal Definition

```typescript
type Phase = "plan" | "approve" | "implement" | "verify" | "deliver";

type Transition = {
  from: Phase;
  to: Phase;
  trigger: "auto" | "human_approval" | "evaluation_passed" | "timeout";
  guard?: (context: Context) => boolean; // transition condition
};

type StateMachine = {
  phases: Record<Phase, SessionConfig>; // each phase has its own configuration
  transitions: Transition[];
  currentPhase: Phase;
  advance: (trigger: string) => Phase; // move to next phase
};
```

Each phase is essentially a different [Session](../patterns/session.md) configuration — different tools, different system prompt, different output target. The StateMachine manages the progression between these configurations according to explicit rules.

---

## Full Workflow Example

```typescript
const developmentWorkflow: StateMachine = {
  phases: {
    plan: {
      tools: [fileRead, grep, glob], // read-only
      system: "Explore the code, propose a plan",
      output: "github_comment", // plan goes to issue comment
    },
    approve: {
      tools: [], // agent doesn't work
      trigger: "human_approval", // waits for human comment
    },
    implement: {
      tools: [fileRead, fileWrite, bash], // full access
      system: "Implement according to plan",
      output: "git_branch",
    },
    verify: {
      tools: [bash, screenshot],
      system: "Run tests, check visuals",
      evaluator: visualEvaluator, // Evaluator as part of phase
      onFail: "implement", // return to implementation
    },
    deliver: {
      tools: [bash], // gh pr create
      output: "github_pr",
    },
  },
  transitions: [
    { from: "plan", to: "approve", trigger: "auto" },
    { from: "approve", to: "implement", trigger: "human_approval" },
    { from: "implement", to: "verify", trigger: "auto" },
    { from: "verify", to: "deliver", trigger: "evaluation_passed" },
    { from: "verify", to: "implement", trigger: "evaluation_failed" },
  ],
};
```

Claude Code plan mode is a degenerate StateMachine with two states (plan → implement). GitHub Action runs are single-pass with no states. But a full workflow requires an arbitrary graph with branching, rollbacks, and human checkpoints.

---

## Implementation Approaches: Graph vs. Imperative

There are two fundamentally different ways to implement a StateMachine:

**Declarative graph** (e.g., LangGraph, Google ADK): Define nodes and edges, compile to a runnable. The topology is declared at build time. Transitions are explicit edges. Conditional routing uses edge condition functions.

**Imperative workflow** (e.g., durable execution engines, raw code): Write regular code with explicit phase tracking. The topology emerges from control flow. Transitions are function calls or state updates.

Both implement the same abstraction — phases, transitions, guards. The trade-offs:

| Property        | Graph                                            | Imperative                     |
| --------------- | ------------------------------------------------ | ------------------------------ |
| Visualization   | Built-in — the graph IS the diagram              | Requires separate tooling      |
| Static analysis | Possible — can verify reachability, dead states  | Hard — topology is dynamic     |
| Dynamic routing | Awkward — requires workarounds for runtime paths | Natural — just write `if/else` |
| Complex logic   | Forced into edge condition functions             | Regular code                   |
| Tooling         | Strong — visual editors, debuggers               | Weak — it's just code          |

The natural split: graph-based for agent-internal routing (fixed topology, benefits from visualization). Imperative for system-level orchestration (dynamic topology, benefits from code flexibility).

---

## Checkpointing and Durability

StateMachine phases are natural checkpointing boundaries. If the system crashes mid-workflow, you want to resume from the last completed phase rather than restarting from scratch.

Checkpointing granularity matters:

- **Per-phase**: Completed phases survive a crash. Partial work inside a phase is lost — the phase re-executes from the start on resume. Requires phases to be idempotent.
- **Per-activity**: Each unit of work within a phase survives independently. Finer recovery, but heavier infrastructure.

The granularity determines how much work you lose on failure and what idempotency guarantees your code must provide. Choose based on the cost of re-execution: if a phase takes 5 seconds of LLM calls, per-phase is fine. If a phase takes 30 minutes of API calls and data processing, per-activity may be necessary.

---

## Human-in-the-Loop as Interrupt/Resume

Human checkpoints in StateMachine follow a universal mechanism: **pause → persist state → wait → resume**.

```typescript
// The human-in-the-loop transition
{
  from: "plan",
  to: "implement",
  trigger: "human_approval",
  // Mechanism: serialize current state, suspend execution,
  // wait for external signal, deserialize state, continue
}
```

On resume, the system must decide what re-executes. If the entire phase re-executes (as in some graph-based implementations), the phase must be idempotent — calling the same APIs twice must produce the same result. If only the continuation after the pause point executes, the checkpoint must capture the exact execution state.

The mechanism is always the same; the implementation details (what re-executes, what must be idempotent) determine correctness.

---

## The Cognitive / Execution Layer Split

StateMachine phases expose a fundamental divide in agent systems:

| Layer         | What it does                                  | Hard problem                       |
| ------------- | --------------------------------------------- | ---------------------------------- |
| **Cognitive** | LLM calls, tool selection, reasoning, routing | Non-determinism, prompt iteration  |
| **Execution** | API calls, database writes, payments          | Reliability, retries, compensation |

A phase that involves only LLM reasoning (planning, analysis) has different infrastructure needs than a phase that involves real-world actions (booking, payment, deployment). The cognitive layer needs prompt iteration and evaluation. The execution layer needs retries, compensation logic, and idempotency guarantees.

This determines what infrastructure a system needs:

- **Cognitively complex, execution-simple** (chatbots, research agents, coding assistants) → a cognitive framework managing both layers is sufficient
- **Cognitively simple, execution-complex** (booking agents, payment flows) → a durable execution engine is sufficient
- **Both complex** → the layers compose: cognitive framework manages reasoning, execution phases invoke durable workflows for dangerous operations

**The durability principle**: durability boundaries should match blast radius. An LLM hallucinating costs a retry. A double-charged credit card costs money and trust. Match reliability machinery to consequence of failure.

---

## Concern-Bundling vs. Separation

Some frameworks (e.g., [LangGraph](../case-studies/langgraph.md)) bundle orchestration (graph routing, checkpointing), state management (shared state + persistence), and execution (tool use, business logic) into a single library. The alternative is full separation: dedicated orchestration, dedicated state management, dedicated execution engines.

Bundling works well for **cognitively complex, execution-simple systems** — when there's nothing dangerous to make durable (no payments, no external API calls requiring compensation), a separate orchestrator is overhead. Separation becomes necessary when execution complexity grows: system-level scheduling, cross-service coordination, or durable execution guarantees on business operations.

The correct choice depends on where the system sits on the cognitive/execution complexity spectrum. Neither approach is universally right — but knowing which trade-off you're making prevents architectural surprises as the system grows.

---

## Anti-Pattern: Prompt-Enforced Orchestration

A common shortcut: encode phase ordering entirely in the orchestrator's system prompt instead of code.

```
# lead_agent.txt (prompt excerpt)
STEP 2: SPAWN RESEARCHER SUBAGENTS (IN PARALLEL)
STEP 3: WAIT FOR RESEARCH COMPLETION
STEP 4: SPAWN DATA-ANALYST SUBAGENT
STEP 5: SPAWN REPORT-WRITER SUBAGENT
```

This is a degenerate StateMachine — all transitions are "auto," there are no guards, no verification that the previous phase succeeded. The LLM's sequential reasoning is the only thing enforcing order.

**Failure modes**:

- Model skips a phase (decides analyst isn't needed) — nothing catches it
- Phase produces empty or garbage output — pipeline continues with it
- Model reorders phases — no code-level constraint prevents it

A subtler variant appears in graph-based frameworks: conditional edges decided by LLM calls without state guards.

```typescript
// Looks like code-level routing, but is actually prompt-enforced:
graph.add_conditional_edges("verify", (state) => llm("should we proceed?"));
```

The guard is non-deterministic. There's no structural guarantee the LLM routes correctly. For production systems, edge conditions should check state (data-driven), not ask the LLM (hope-driven).

**When prompt-enforced orchestration is acceptable**: Prototyping, demos, low-stakes pipelines where approximate ordering is fine.
**When it breaks**: Anything where you need guarantees. Add code-level guards:

```typescript
{ from: "research", to: "analyze",
  guard: (ctx) => glob("files/research_notes/*.md").length >= 2,
  trigger: "auto" }
```

---

## Related

- [Session](../patterns/session.md) — a single continuous conversation; StateMachine chains multiple Sessions with different configurations
- [Guardrail](guardrail.md) — constrains actions within a phase; StateMachine constrains transitions between phases
- [Evaluator](../patterns/evaluator.md) — often used as a transition trigger (`evaluation_passed` / `evaluation_failed`)
- [Deployment](../patterns/deployment.md) — StateMachine is a key component of production deployments
- [LangGraph case study](../case-studies/langgraph.md) — a framework built around StateMachine as a product
