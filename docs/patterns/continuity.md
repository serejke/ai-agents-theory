# Continuity

Continuity is the protocol that enables coherent agent progress across multiple context windows. It is what separates "start from zero every session" from "pick up where the last session left off" — the difference between an agent that can handle a 30-minute task and one that can handle a 30-day project.

**Composition**: [Memory](../primitives/memory.md) (ground truth + progress log) + [ContextProvider](context.md) (startup sequence) + [Guardrail](../primitives/guardrail.md) (clean state enforcement) + [Tool](../primitives/tool.md) (environment bootstrap).

## Why It Matters

A [Session](session.md) is bounded by the [LLM](../primitives/llm.md)'s context window. When a task exceeds that window — and most real projects do — the agent needs multiple sessions. Without explicit continuity mechanisms, each new session faces two failure modes:

1. **Premature completion.** The agent looks around, sees code that exists, and concludes the job is done. Partially implemented features look like fully implemented features from inside a fresh context window. Without a ground truth document that defines "done," the agent has no way to know the difference.

2. **Archaeological waste.** The agent spends its context budget figuring out what was done, what state things are in, and how to start the environment. This orientation work is identical across every session and produces no forward progress. Without a progress log and environment bootstrap, each session pays this tax from scratch.

Both failures share a root cause: the agent has no persistent, structured understanding of the project's state that survives the context window boundary.

## Formal Definition

```typescript
type Continuity = {
  groundTruth: GroundTruth; // what "done" means — survives all sessions
  progressLog: ProgressLog; // what was done, what's next — updated each session
  bootstrap: Tool; // reproducible environment setup
  startupSequence: ContextProvider; // orientation protocol at session start
  cleanState: Guardrail; // enforced handoff discipline at session end
};
```

---

## Five Components, Five Failure Modes

Each component addresses a specific failure mode. Remove any one and the corresponding failure returns.

### Ground Truth — Prevents Premature Completion

A structured, enumerated list of verifiable completion criteria. Each item describes a user-visible behavior that can be tested end-to-end, with an explicit status the agent updates only after verification.

```typescript
type GroundTruth = {
  items: Array<{
    description: string; // "user can open a new chat, type a query, see a response"
    steps: string[]; // verification steps
    status: "done" | "todo"; // updated only after end-to-end verification
  }>;
  format: "json"; // rigid structure resists casual modification
};
```

The ground truth is stored as JSON rather than Markdown. JSON's rigid structure resists casual editing — the agent treats it as data to update carefully, not prose to rewrite. This is a deliberate cognitive affordance: the format reinforces the intended interaction pattern.

Without ground truth, the agent infers completeness from the code. This inference is unreliable — code can exist that isn't functional, functionality can exist that's incomplete. Ground truth makes completeness explicit and unambiguous.

### Progress Log — Prevents Archaeological Waste

An append-only log that agents update at the end of every session, documenting what they worked on, what they completed, and what state they left things in.

```typescript
type ProgressLog = {
  entries: Array<{
    session: string; // session identifier
    worked_on: string; // what was attempted
    completed: string[]; // what was finished
    state: string; // what state things are in
    next: string; // recommended next step
  }>;
};
```

Combined with git history, the progress log gives every future session a fast orientation mechanism. A new session reads the latest entry and immediately knows: what was done last, what's next, what state the environment is in. No archaeology required.

### Bootstrap — Prevents Setup Tax

A reproducible environment setup mechanism (script, tool, or procedure) that brings the development environment to a working state in one step.

```typescript
type Bootstrap = Tool & {
  name: "bootstrap";
  // Starts servers, sets up databases, installs dependencies.
  // Identical across all sessions — no agent decision-making required.
  execute: () => Promise<{ output: "environment ready" }>;
};
```

Every session that follows the first can begin by running the bootstrap rather than spending tokens figuring out how to start servers, set up databases, and get the application into a testable state. The tokens saved on environment setup in every session accumulate significantly over a long project.

### Startup Sequence — Prevents Disoriented Work

A fixed sequence of context-gathering steps that every session executes before beginning new work:

```typescript
const startupSequence: ContextProvider = () => {
  // 1. Read progress log — what was done last
  // 2. Read ground truth — what's done vs. remaining
  // 3. Check git log — recent changes
  // 4. Run bootstrap — start environment
  // 5. Run basic verification — confirm environment works
  // Only then: choose next task and begin work
};
```

The startup sequence is a [ContextProvider](context.md) that assembles orientation context from the other continuity components. If the verification step reveals the environment is broken, the agent fixes existing breakage before touching anything new — preventing the compounding problem where new work is built on a broken foundation.

### Clean State — Prevents Messy Handoffs

A [Guardrail](../primitives/guardrail.md) that enforces handoff discipline at the end of every session. The agent cannot consider its work done until:

```typescript
const cleanState: Guardrail = {
  post: (action, result) => {
    if (action.name === "session_end") {
      if (!gitStatusClean()) return { block: "uncommitted changes" };
      if (!progressLogUpdated()) return { block: "progress log not updated" };
      if (!buildPasses()) return { block: "build is broken" };
    }
    return { allow: true };
  },
};
```

This is a Guardrail, not a suggestion in the system prompt. Anything the LLM can ignore, the LLM will eventually ignore. Structural enforcement prevents the failure mode where agents leave half-implemented features, broken tests, and undocumented changes for the next session to untangle.

Git commits serve double duty here: they're both a documentation mechanism (descriptive commit messages) and a recovery mechanism (revert to last known-good state on failure).

---

## Continuity vs. Memory

Continuity composes [Memory](../primitives/memory.md) with other primitives to solve a specific problem. The relationship:

| Concern       | Memory                        | Continuity                                |
| ------------- | ----------------------------- | ----------------------------------------- |
| Scope         | Cross-session knowledge       | Cross-session task progress               |
| Content       | Decisions, preferences, facts | Ground truth, progress, environment state |
| Write trigger | Extraction from interaction   | Session boundary (start and end)          |
| Read trigger  | Relevance to current task     | Every session start (fixed protocol)      |
| Without it    | Agent forgets preferences     | Agent loses track of multi-session work   |

Memory answers "what does the agent know from past experience?" Continuity answers "where is the agent in a multi-session task?" Both are cross-session persistence, but they solve different problems and compose independently — an agent can have Memory without Continuity (remembers preferences but can't track multi-session progress) or Continuity without Memory (tracks progress but forgets that you prefer TypeScript strict mode).

---

## When Continuity Matters

Continuity is unnecessary for tasks that fit within a single context window. For a quick bug fix, a code review, or a short conversation, [Session](session.md) history is sufficient.

Continuity becomes essential when:

- **The task exceeds one context window.** Building a feature, implementing a specification, any project that takes more than one session.
- **Multiple agents work on the same project.** Each agent needs orientation — not just what it did last time, but what all agents have done. Ground truth and progress logs serve as the coordination surface.
- **The project spans days or weeks.** Session history is volatile. Without Continuity, a Monday session has no connection to Friday's session.

---

## Related

- [Session](session.md) — within-session history; Continuity bridges across sessions
- [Memory](../primitives/memory.md) — cross-session knowledge; Continuity is the task-progress specialization
- [ContextProvider](context.md) — the startup sequence is a ContextProvider composition
- [Guardrail](../primitives/guardrail.md) — clean state enforcement is a Guardrail
- [Workspace](workspace.md) — ground truth and progress logs live in the workspace as files
