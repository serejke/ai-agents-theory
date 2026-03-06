# Agent Patterns — Planner, Router, Evaluator, Guardrails, State Machine

Analysis of the remaining elements from the primitives table. Key question: which are true new primitives (irreducible building blocks), and which are patterns (typical compositions of existing primitives)?

Analogy from FP: `map`, `filter`, `reduce` are not new language constructs but patterns over recursion and functions. But that doesn't make them less important — patterns are what you use every day.

Related: [Agent Primitives](agent-primitives.md), [Memory](memory.md), [Channel](channel.md)

---

## Planner — Pattern, Not Primitive

```typescript
// Planner is just an AgentLoop with a specific system prompt and output format
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

This is **not a new primitive** — it's an AgentLoop with a specific configuration (read-only tools + system prompt requiring structured output). Claude Code plan mode, Devin planning phase, Codex reasoning — all do the same thing.

**Universal pattern**: any system where the agent first thinks, then acts — uses Planner.

---

## Router — Pattern

```typescript
// Router — an ordinary function choosing a configuration
type Router = (input: string, context: Context) => SessionConfig;

const router: Router = (input, context) => {
  // Via heuristics (cheap):
  if (isSimpleQuestion(input)) return { llm: haiku, tools: [fileRead] };
  if (needsCodeChanges(input)) return { llm: sonnet, tools: allTools };
  if (isArchitectural(input)) return { llm: opus, tools: readOnlyTools };
};

// Or Router via LLM (expensive, but flexible):
const llmRouter: Router = async (input) => {
  const decision = await llm("classify", [{ role: "user", content: input }]);
  return configMap[decision]; // "simple" | "code" | "architecture"
};
```

Claude Code does this implicitly — chooses haiku for sub-agents (Task tool), sonnet for the main loop. But as an explicit primitive it's not exposed.

**Universal pattern**: all multi-model systems use it.

### Variant: Prompt-as-Router (LLM routing via agent descriptions)

In Agent-as-Tool systems, you can encode routing logic as natural language descriptions on each subagent definition:

```typescript
const agents = {
  researcher: {
    description: "Use when you need to gather research information...",
    // ...
  },
  "data-analyst": {
    description: "Use AFTER researchers have completed their work...",
    // ...
  },
};
// The orchestrator LLM reads descriptions and decides which agent to invoke.
```

The word "AFTER" in the data-analyst description does the work of a state transition guard. Preconditions, postconditions, and capability boundaries are encoded in natural language.

**Pros**: Zero routing code. Flexible — LLM adapts to context. Easy to add new agents.
**Cons**: Non-deterministic — model may skip agents or invoke them out of order. Can't audit routing logic. No guarantees. Not suitable when ordering must be strict.

**Applicability**: Prototypes, demos, systems where approximate routing is acceptable. For production pipelines with strict phase ordering, use explicit StateMachine transitions instead.

---

## Evaluator — Pattern, But on the Edge of Being a Primitive

```typescript
// Evaluator = LLM call (or AgentLoop) focused on assessment
type Evaluator = (
  task: string,      // what was wanted
  result: string,    // what was produced
  evidence?: Evidence, // screenshot, tests, diff
) => Promise<Verdict>;

type Verdict = {
  pass: boolean;
  issues: string[];      // what's wrong
  suggestions: string[]; // how to fix
};

// Simple evaluator — one LLM call:
const codeEvaluator: Evaluator = async (task, result) => {
  return await llm("evaluate", [
    {
      role: "user",
      content: `Task: ${task}\nResult: ${result}\nDoes it match?`,
    },
  ]);
};

// Visual evaluator — multimodal:
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

Why "on the edge of being a primitive"? Because Evaluator creates a **feedback loop** — a closed cycle that qualitatively changes system behavior:

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

Formally — it's a composition of LLM + loop. But it produces **new behavior** (self-correction) that doesn't exist in any of the base primitives individually. All systems that "iterate until result" use it.

**Universal pattern**: not tied to any specific tool.

---

## Guardrails — True Separate Primitive

Guardrails are **not AgentLoop and not Tool**. They're middleware — a wrapper that filters actions before and after execution:

```typescript
type Guardrail = {
  // Pre: check BEFORE executing an action
  pre?: (action: ToolCall) => Allow | Deny | AskUser;

  // Post: check AFTER execution
  post?: (action: ToolCall, result: ToolResult) => Allow | Redact | Block;
};

// Examples:
const noForcePush: Guardrail = {
  pre: (action) => {
    if (
      action.name === "bash" &&
      action.params.command.includes("push --force")
    )
      return { deny: "force push is forbidden" };
    return { allow: true };
  },
};

const noSecrets: Guardrail = {
  post: (action, result) => {
    if (containsApiKey(result.output))
      return { redact: maskSecrets(result.output) };
    return { allow: true };
  },
};

// Application — wraps a Tool
const guardedTool = (tool: Tool, guardrails: Guardrail[]): Tool => ({
  ...tool,
  execute: async (params) => {
    for (const g of guardrails) {
      const check = g.pre?.({ name: tool.name, params });
      if (check?.deny) throw new Error(check.deny);
      if (check?.askUser) await confirmWithUser(check.askUser);
    }
    const result = await tool.execute(params);
    for (const g of guardrails) {
      const check = g.post?.({ name: tool.name, params }, result);
      if (check?.redact) return check.redact;
      if (check?.block) throw new Error(check.block);
    }
    return result;
  },
});
```

Why is this a **true primitive**, not a pattern? Because Guardrail is the only place in the system that is **not controlled by the LLM**. Everything else — system prompt, tools, memory — the LLM can ignore or interpret its own way. Guardrail is a hard constraint, enforced in code, beyond the model's reach.

In Claude Code this is: permission system ("can it write files?"), safety rules, restricted commands. But in Claude Code guardrails are hardcoded, whereas ideally this should be a composable layer — so the user can add their own constraints.

**Universal primitive**: any production system needs guardrails.

### Two Modes: Observe and Control

The same hook/middleware infrastructure serves two purposes:

- **Observe mode**: Log every tool call for debugging, audit, cost tracking. Always returns `allow`. Example: hooks that log to transcript + JSONL but never deny.
- **Control mode**: Enforce constraints. Deny dangerous actions, redact sensitive output, validate quality before allowing writes.

The infrastructure is identical (pre/post callbacks wrapping tool execution). The difference is in what the callback returns. Build hooks once, use for both.

---

## State Machine — True Primitive (and the Most Underrated)

```typescript
type Phase = "plan" | "approve" | "implement" | "verify" | "deliver";

type Transition = {
  from: Phase;
  to: Phase;
  trigger: "auto" | "human_approval" | "evaluation_passed" | "timeout";
  guard?: (context: Context) => boolean; // transition condition
};

type StateMachine = {
  phases: Record<Phase, SessionConfig>; // each phase has its own config
  transitions: Transition[];
  currentPhase: Phase;

  advance: (trigger: string) => Phase; // move to next phase
};
```

Full workflow example:

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

Why is this a **primitive**, not a pattern? Because State Machine introduces a concept absent from base blocks: **phases with different capabilities and transition rules between them**. Session is one continuous conversation. State Machine is a chain of Sessions with different configurations and explicit control points (including human-in-the-loop checkpoints).

Claude Code plan mode is a degenerate State Machine with two states (plan -> implement). GitHub Action is a single pass with no states. But a full workflow requires an arbitrary graph with branching, rollbacks, and human checkpoints.

This is a **universal primitive**. And arguably the main missing piece for transitioning from "agent in terminal" to "agent as CI/CD pipeline".

### Anti-pattern: Prompt-Enforced Orchestration (StateMachine without guards)

A common shortcut: encode the phase ordering entirely in the orchestrator's system prompt instead of code. Example from a multi-agent research pipeline:

```
# lead_agent.txt (prompt excerpt)
STEP 2: SPAWN RESEARCHER SUBAGENTS (IN PARALLEL)
STEP 3: WAIT FOR RESEARCH COMPLETION
STEP 4: SPAWN DATA-ANALYST SUBAGENT
STEP 5: SPAWN REPORT-WRITER SUBAGENT
```

This is a **degenerate StateMachine** — all transitions are "auto", there are no guards, no verification that the previous phase succeeded. The LLM's sequential reasoning is the only thing enforcing order.

**Failure modes**:
- Model skips a phase (decides analyst isn't needed) — nothing catches it
- Phase produces empty/garbage output — pipeline continues with it
- Model reorders phases — no code-level constraint prevents it

**When it's fine**: Prototyping, demos, low-stakes pipelines.
**When it breaks**: Anything where you need guarantees. Add code-level guards:

```typescript
// What production needs:
{ from: "research", to: "analyze",
  guard: (ctx) => glob("files/research_notes/*.md").length >= 2,
  trigger: "auto" }
```


