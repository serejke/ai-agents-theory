# Session

A Session wraps an [AgentLoop](agent-loop.md) with persistent conversation history, turning a stateless computation into a stateful conversational agent. It is what makes multi-turn interaction possible.

**Composition**: AgentLoop + history (Message[]).

## Why It Matters

[AgentLoop](agent-loop.md) takes messages as input and produces events as output, but doesn't maintain state between invocations. Each call is independent — the loop doesn't know what happened in the previous turn.

A Session adds what AgentLoop lacks: an accumulating `history` array that grows with each turn. When the user sends a message, it's appended to history. When the agent responds, the response is appended to history. The next LLM call sees the full conversation so far. This is what makes "remember what I said earlier" work.

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

The difference in the model is **zero**. The only difference is in the I/O interface. This is a direct consequence of the compositional model: Session is Session regardless of what triggers it. The trigger is a separate concern — see [Deployment](deployment.md).

---

## Session as Working Memory

In cognitive terms, Session history is **working memory** — the active context the agent uses for the current task. It's fast (always available, no retrieval needed), limited (bounded by context window), and volatile (lost when the session ends).

This contrasts with [Memory](../primitives/memory.md) (the primitive), which is **long-term memory** — persistent across sessions, potentially unlimited, but requires explicit storage and retrieval. The relationship:

- **Session history**: what the agent knows about _this_ conversation (working memory)
- **Memory primitive**: what the agent knows from _past_ conversations (long-term memory)

The gap between them is where knowledge is lost. Everything discussed in a session that isn't extracted into Memory disappears when the session ends. Closing this gap — automatic extraction of decisions and preferences from session history into persistent memory — is the main unsolved problem in the [Memory](../primitives/memory.md) primitive.

---

## Related

- [AgentLoop](agent-loop.md) — the computation pattern that Session wraps with state
- [LLM](../primitives/llm.md) — the stateless function whose context window bounds Session history
- [Memory](../primitives/memory.md) — cross-session persistence; complements Session's within-session history
- [ContextProvider](context.md) — injects knowledge into the Session's system prompt
- [Deployment](deployment.md) — how Sessions are triggered and where their output goes
