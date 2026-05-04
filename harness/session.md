# Session

A Session wraps an [AgentLoop](agent-loop.md) with persistent conversation history, turning a stateless computation into a stateful conversational agent. It is the layer of the harness that makes multi-turn interaction possible.

**Composition**: AgentLoop + history (Message[]).

## Why It Matters

[AgentLoop](agent-loop.md) takes messages as input and produces events as output, but doesn't maintain state between invocations. Each call is independent — the loop doesn't know what happened in the previous turn.

A Session adds what AgentLoop lacks: an accumulating `history` array that grows with each turn. When the user sends a message, it's appended to history. When the agent responds, the response is appended to history. On the next LLM call, a snapshot of this history is assembled into the [Prompt](../primitives/prompt.md)'s `messages` slot — Session is the owner, Prompt is the per-call view. This is what makes "remember what I said earlier" work.

Session is harness, not a compositional choice. As soon as an agent runs the loop more than once, it has a Session — even if the author never writes the word. A single-shot invocation is a degenerate Session (history of length 1); a long-running chat is a Session whose history grows until it hits the context window.

## Formal Definition

```typescript
type Session = {
  history: Message[]; // accumulator — grows with each turn
  loop: AgentLoop;

  // send — the only method. Adds message, calls loop,
  // accumulates response in history
  send: (input: string) => AsyncGenerator<Event>;
};
```

**Limitation**: history is bounded by the LLM's context window (~200K tokens). When it overflows, the session "dies" or loses the beginning. This is a hard limit — not a design choice but a physical constraint of the underlying [LLM](../primitives/llm.md) primitive.

---

## Claude Code as a Session

A Claude Code CLI window is literally a Session:

```typescript
const claudeCodeSession = createSession({
  llm: claude_sonnet,
  tools: [fileRead, fileWrite, bash, grep, glob, webFetch, ...],
  system: buildSystem(
    coreInstructions,      // "you are Claude Code, you help with code"
    claudeMd,              // .claude/CLAUDE.md
    projectContext,         // project structure, git status
    planModeRules,         // plan mode rules
  ),
});
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

The difference in the model is **zero**. The only difference is in the I/O interface. This is a direct consequence of the compositional model: Session is Session regardless of what wraps it. The frontend — stdin, an SDK call, an HTTP handler — is a separate concern; see [Agent](agent.md).

---

## Session as Working Memory

In cognitive terms, Session history is **working memory** — the active context the agent uses for the current task. It's fast (always available, no retrieval needed), limited (bounded by context window), and volatile (lost when the session ends).

This contrasts with [Memory](../patterns/memory.md), which is **long-term memory** — persistent across sessions, potentially unlimited, but requires explicit storage and retrieval. The relationship:

- **Session history**: what the agent knows about _this_ conversation (working memory)
- **Memory**: what the agent knows from _past_ conversations (long-term memory)

The gap between them is where knowledge is lost. Everything discussed in a session that isn't extracted into Memory disappears when the session ends. Closing this gap — automatic extraction of decisions and preferences from session history into persistent memory — is the main unsolved problem in the [Memory](../patterns/memory.md) pattern.

---

## Session-Graphs

Sessions are linear by definition, but multiple sessions can share a common prefix — a forked session's first N messages are identical to another's. Most systems hide this relation: forking means a full copy, resuming means loading one session in isolation. A **session-graph** makes the shared-prefix relation first-class — sessions are stored as parent-linked nodes, and a pointer selects which leaf is active. Branching becomes a pointer move rather than a copy; switching between explored alternatives is cheap.

This is a [Workspace](../patterns/workspace.md)-shaped structure _over_ Sessions, not a change to Session itself. Each leaf is still a linear history, and the LLM always sees a flat root-to-leaf slice. What becomes navigable is the graph itself — nodes are conversation states, edges are parent links. For workflows that rely on exploration ("try three approaches, keep the one that worked") the session-graph is a useful structure to expose; for conventional turn-by-turn chat it is unnecessary. The [PI case study](../case-studies/pi.md) is a concrete implementation.

---

## Related

- [AgentLoop](agent-loop.md) — the computation that Session wraps with state
- [LLM](../primitives/llm.md) — the stateless function whose context window bounds Session history
- [Memory](../patterns/memory.md) — cross-session persistence; complements Session's within-session history
- [PromptLoading](../patterns/prompt-loading.md) — injects knowledge into the Session's system prompt
- [Continuity](../patterns/continuity.md) — the protocol for coherent progress across multiple Sessions
- [Agent](agent.md) — how Sessions get wrapped in a Frontend and pinned to an Environment
