# Agent Primitives — A Functional Approach

An attempt to formalize the building blocks of agent systems in a functional programming style — to understand what bricks tools like Claude Code, GitHub Actions, Devin, etc. are assembled from, and what new systems can be built from them.

Notation: TypeScript-like types.

---

## Layer 0: Atomic Primitives

```typescript
// Message — unit of communication
type Message =
  | { role: "user"; content: string }
  | { role: "assistant"; content: string; toolCalls?: ToolCall[] }
  | { role: "tool_result"; callId: string; result: string };

// LLM — pure function (stateless). All state is in the arguments.
type LLM = (
  system: string,
  messages: Message[],
) => AsyncStream<TextDelta | ToolCall>;

// Tool — action handler. The only place with side effects.
type Tool = {
  name: string;
  schema: JSONSchema;
  execute: (params: unknown) => Promise<ToolResult>; // side effect!
};
```

**Key observation:** LLM is stateless. It has no memory between calls. Everything it "remembers" is what was passed in `messages`. This is fundamental: any agent "memory" is an external construct, not a property of the model.

---

## Layer 1: Agent Loop — First Composition

```typescript
// AgentLoop closes LLM + Tools into a recursive cycle
type AgentLoop = (
  llm: LLM,
  tools: Tool[],
  system: string,
) => (messages: Message[]) => AsyncGenerator<Event>;

// Semantics:
// 1. Call llm(system, messages)
// 2. If response is text -> yield TextDelta, return
// 3. If response is toolCalls -> execute tools, append results
//    to messages, goto 1
```

This is the **minimal agent**. Copilot, ChatGPT, Claude.ai, any "AI assistant" — all implement this primitive. The difference between them is in the set of `tools` and contents of `system`.

---

## Layer 2: Context — What the Agent Knows

```typescript
// ContextProvider — a function that gathers context
// and turns it into part of the system prompt
type ContextProvider = () => string | Message[];

// Examples:
const claudeMd: ContextProvider = () => readFile(".claude/CLAUDE.md");
const repoStructure: ContextProvider = () => glob("**/*.ts").join("\n");
const gitHistory: ContextProvider = () => exec("git log --oneline -20");

// Composite system prompt
const buildSystem = (...providers: ContextProvider[]): string =>
  providers.map((p) => p()).join("\n\n");
```

`CLAUDE.md` is precisely a ContextProvider. Not "memory" in the strict sense (the agent doesn't automatically write to it between sessions), but a **knowledge injection** into a stateless function.

### Eager vs Lazy Context

ContextProviders have two loading strategies:

```typescript
// Eager — loaded at session start, always in context window
type EagerContext = () => string; // CLAUDE.md, git status, project structure

// Lazy — loaded on-demand when the agent decides it needs it
// Delivered via a Tool (e.g., Claude Code's Skill tool)
type LazyContext = {
  name: string;
  description: string; // agent reads this to decide whether to invoke
  load: () => string; // injected only when invoked
};
```

**Eager** works while context is small. **Lazy** scales — the agent pulls domain knowledge (PDF generation guides, API references, coding standards) only when relevant. Pattern: keep system prompts small, use lazy injection for domain knowledge. Example: Claude Code's Skill tool loads reference guides on-demand rather than embedding them in the system prompt.

---

## Layer 3: Session — Stateful Wrapper

```typescript
// Session adds what AgentLoop lacks: state (history)
type Session = {
  history: Message[]; // accumulator — grows with each turn
  loop: AgentLoop;

  // send — the only method. Adds message, calls loop,
  // accumulates response in history
  send: (input: string) => AsyncGenerator<Event>;
};

// Limitation: history is bounded by context window (~200K tokens).
// When it overflows — session "dies" or loses the beginning.
```

**Claude Code CLI window = Session.** Literally:

```typescript
const claudeCodeSession = createSession({
  llm: claude_sonnet,
  tools: [fileRead, fileWrite, bash, grep, glob, webFetch, ...],
  system: buildSystem(
    coreInstructions,      // "you are Claude Code, you help with code"
    claudeMd,              // .claude/CLAUDE.md
    projectContext,         // project structure, git status
    planModeRules,         // plan mode rules
  )
})
```

**Claude Code SDK** — the same Session, but `send()` is called from code, not from stdin:

```typescript
// CLI:
readline.on("line", (input) => {
  session.send(input);
});

// SDK:
const result = await session.send("add tests for the auth module");
```

The difference in the model is **zero**. The only difference is in the I/O interface.

---

## Layer 4: Trigger + Session + Output = Deployment

This layer explains why Claude Code CLI, Claude Code SDK, and Claude Code GitHub Action are the same entity with different "wiring":

```typescript
type Trigger =
  | { type: "cli_stdin"; input: string } // Claude Code CLI
  | { type: "sdk_call"; input: string } // Claude Code SDK
  | { type: "github_issue"; issue: GitHubIssue } // GitHub Action
  | { type: "github_comment"; comment: GitHubComment } // GitHub Action
  | { type: "cron"; schedule: string } // scheduled
  | { type: "webhook"; payload: unknown }; // external signal

type OutputChannel =
  | { type: "cli_stdout" } // prints to terminal
  | { type: "github_pr"; repo: string } // creates PR
  | { type: "github_comment"; issueId: number } // writes comment
  | { type: "file"; path: string }; // writes to file

// "Deployment" — full agent configuration
type AgentDeployment = {
  trigger: Trigger;
  session: Session; // or SessionFactory — new Session per trigger
  output: OutputChannel;
  environment: Environment;
};
```

**Environment** — three layers that don't have to be together (for many tasks only the first is enough):

```typescript
type Environment = {
  code: CodeAccess; // git clone / local files / read-only
  runtime?: Runtime; // npm, node, dev-server, build tools
  os?: OSAccess; // gh, git push, ssh, credentials
};

// "Light" agent — code only (planning, review, analysis)
const lightAgent: Environment = { code: gitClone("repo") };

// "Heavy" agent — full environment (build, test, deploy)
const heavyAgent: Environment = {
  code: gitClone("repo"),
  runtime: container({ image: "node:20", cmd: "npm run dev" }),
  os: shell({ credentials: ["gh_token", "ssh_key"] }),
};
```

---

## Layer 5: Agent Composition

**Agent-as-Tool** — the result of one agent's work becomes a tool result for another:

```typescript
const subAgent: Tool = {
  name: "research_codebase",
  execute: async (params) => {
    const sub = createSession({ llm, tools: [fileRead, grep, glob] });
    const result = await sub.send(params.question);
    return result; // returned as ToolResult in parent session
  },
};
```

This is exactly what Claude Code does with the `Task` tool — spawns a sub-agent (new Session) with a limited set of tools.

**Multi-agent** — multiple sessions + communication channel:

```typescript
type Team = {
  agents: Record<string, Session>;
  mailbox: MessageBus; // agents send messages to each other
  taskList: SharedState; // shared task list
};
```

---

## Final Primitives Map

```
LLM ────────────────── stateless: (system, messages) -> stream
  |
  +-- Tool ──────────── side effect: (params) -> result
  |
  +-- AgentLoop ─────── LLM + Tool[] in recursive cycle
  |    |
  |    +-- ContextProvider -- () -> string (knowledge injection)
  |    |
  |    +-- Session ────── AgentLoop + history (stateful)
  |         |
  |         +-- Trigger ── what starts Session
  |         +-- Output ─── where Session writes
  |         +-- Environment -- code / runtime / OS
  |
  +-- Composition ────── Session as Tool | Team (mailbox + shared state)
```

---

## Updated Classification (after deep-dives)

```
=== PRIMITIVES (irreducible) ===

LLM ──────────────── stateless: (system, messages) -> stream
Tool ─────────────── side effect: (params) -> result
Memory ──────────── persistent read+write context (see [Memory](memory.md))
Guardrail ────────── hard constraint, not controlled by LLM (see [Agent Patterns](agent-patterns.md))
StateMachine ─────── phases + transitions + human checkpoints (see [Agent Patterns](agent-patterns.md))
Channel ──────────── intermediate results between agents within a run (see [Channel](channel.md))

=== PATTERNS (compositions of primitives) ===

AgentLoop ────────── LLM + Tool[] in a cycle          (LLM + Tool)
Session ──────────── AgentLoop + history               (AgentLoop + state)
Planner ──────────── Session with read-only tools      (Session + constraint)
Router ───────────── dispatch by input                 (function)
Evaluator ────────── LLM-as-judge + feedback loop      (LLM + loop)
Workspace ────────── structured data space for agents  (Tool + Channel + Context) (see [Workspace](workspace.md))

=== INFRASTRUCTURE ===

Trigger ──────────── what starts it (CLI / issue / cron / webhook)
Output ───────────── where it writes (stdout / PR / comment)
Environment ──────── code / runtime / OS
```

All primitives and patterns are **universal** — none are specific to Claude Code. Claude Code is one specific assembly:

```typescript
const claudeCode: AgentDeployment = {
  trigger: "cli_stdin",
  session: Session(AgentLoop(claude_sonnet, codeTools, systemPrompt)),
  output: "cli_stdout",
  environment: { code: localFS, runtime: localNode, os: localShell },
  memory: claudeMd, // manual, always_load
  guardrails: [noForcePush, askBeforeWrite, safetyRules],
  stateMachine: twoPhasePlanMode, // plan -> implement (degenerate)
};
```

Codex, Devin, Gemini CLI, Cursor — different assemblies from the same bricks. The difference is in specific parameter values, not architecture.

---

## Primitives That Don't Exist Yet But Will Appear

| Primitive                | Purpose                                    | Example                                                       |
| ------------------------ | ------------------------------------------ | ------------------------------------------------------------- |
| **Memory** (persistent)  | Knowledge persisting between sessions      | Agent remembers decisions from past tasks, author preferences |
| **Planner**              | Task decomposition before AgentLoop        | "Break into subtasks, then execute each"                      |
| **Evaluator / Verifier** | Verification of agent's result             | Screenshot -> vision model -> "does it match the task?"       |
| **Router**               | Choosing which agent/model handles request | Light task -> haiku, heavy -> opus                            |
| **Guardrails**           | Input/output filter                        | "Don't push to main", "don't delete files"                    |
| **State Machine**        | Managing work phases                       | plan -> approve -> implement -> verify -> PR                  |

Especially interesting is **State Machine**: a two-phase workflow ("first propose plan in issue, then code") is a state transition that doesn't exist as an explicit primitive in current systems. Plan mode in Claude Code is an approximation, but hardcoded to two states.
