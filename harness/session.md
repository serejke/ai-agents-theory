# Session

A **Session** is an [AgentLoop](agent-loop.md) that keeps the history of its
conversation. Each message and each output of the agent go into this history,
and the LLM reads the history again on each turn. This part of the harness lets
an agent continue a conversation across many turns.

**Composition**: AgentLoop + history (Message[]).

## Value

<a id="why-it-matters"></a>

The Session gives the agent the history of the conversation. Without it, the
agent does not know about the previous turn. The [AgentLoop](agent-loop.md) gets
messages as input and gives `Event` items as output, but it keeps no data
between two calls. Each call starts again without data from the previous call.

The Session adds a `history` array that becomes larger with each turn:

- When the user sends a message, the Session adds the message to the history.
- When the agent gives an output, the Session adds the output to the history.
- Before the next LLM call, the harness puts a copy of the history into the
  `messages` slot of the [Prompt](../primitives/prompt.md).

The Session holds the history. The Prompt is the view of the history for one
call. Thus the agent knows the previous messages of the user.

The Session is a part of the harness, not an optional pattern. When an agent
runs the loop more than one time, it has a Session, also if its author does not
use the word. A call with one input is a Session with a history of length 1. A
long chat is a Session with a history that becomes larger until it fills the
context window.

## Definition

<a id="formal-definition"></a>

```typescript
type Session = {
  history: Message[]; // grows with each turn
  loop: AgentLoop;

  // send is the only method. It adds the input to history, calls the loop,
  // and adds the response to history.
  send: (input: string) => AsyncGenerator<Event>;
};
```

The context window of the [LLM](../primitives/llm.md) sets the limit of the
history: approximately 200,000 tokens. If the history becomes larger, the
Session stops, or it removes the start of the history. This is a hard limit. The
author of the system does not select it. The Session uses the LLM, and the limit
comes from the LLM.

## Claude Code as a Session

Each run of Claude Code in a terminal is a Session:

```typescript
const claudeCodeSession = createSession({
  llm: claude_sonnet,
  tools: [fileRead, fileWrite, bash, grep, glob, webFetch, ...],
  system: buildSystem(
    coreInstructions,      // "you are Claude Code, you help with code"
    claudeMd,              // .claude/CLAUDE.md
    projectContext,        // project structure, git status
    planModeRules,         // plan mode rules
  ),
});
```

The Claude Code SDK uses the same Session. But code calls `send()`, not a person
who writes in the terminal:

```typescript
// CLI:
readline.on("line", (input) => {
  session.send(input);
});

// SDK:
const result = await session.send("add tests for the auth module");
```

The model of the two examples is the same. Only the interface for the input and
the output is different. This comes from the composition: a Session stays the
same Session for each frontend around it. The frontend is a different part of
the system, for example the standard input, an SDK call or an HTTP handler. The
[Agent](agent.md) document tells about the frontend.

<a id="session-as-working-memory"></a>

## Session history and Memory

In terms of the memory of a person, the Session history is the **working
memory** of the agent: the active context of the task. The Session history has
these properties:

- It is fast. It is always available, and the agent does not use a tool to find
  it.
- It is small. The context window sets its maximum dimension.
- It does not stay. When the Session stops, its history is not available.

[Memory](../patterns/memory.md) is the **long-term memory** of the agent. It
stays across Sessions, and it can be very large. But the agent must write each
fact into it and then find the fact again with a tool.

- **Session history**: the knowledge of the agent about _this_ conversation
  (working memory).
- **Memory**: the knowledge of the agent from _previous_ conversations
  (long-term memory).

Knowledge does not always go from the Session history into Memory. If the agent
does not move a fact from the Session history into Memory, the fact is not
available after the Session stops. The automatic movement of decisions and user
selections from the Session history into Memory is the most important open
problem of the [Memory](../patterns/memory.md) pattern.

<a id="session-graphs"></a>

## Session graphs

A **session graph** keeps two or more Sessions that start with the same messages
in one tree. Each Session is linear. A **fork** of a Session has the same first
messages as the Session that it comes from.

Most systems do not show this relation. A fork makes a full copy, and a system
that continues a Session loads only that Session. A session graph shows the same
start directly:

- It stores the Sessions as nodes with links to the previous node.
- A pointer selects the active **leaf**: the last node of one Session.
- A new branch only moves the pointer, and it does not make a copy. Thus the
  user can go between the alternatives quickly.

The session graph is a structure of the [Workspace](../patterns/workspace.md)
pattern _above_ the Sessions. It does not change the Session. Each leaf is
always a linear history. The LLM always sees a flat part of the graph from the
root to the leaf. The user can move through the graph: the nodes are conditions
of the conversation, and the edges are links to the previous node.

A session graph helps in a workflow that tries alternatives, for example "try
three approaches, keep the one that worked". A usual chat with one turn after
the other does not use it. The [PI case study](../case-studies/pi.md) shows an
implementation.

## Related

- [AgentLoop](agent-loop.md): the loop that the Session gives a history.
- [LLM](../primitives/llm.md): the function without memory. Its context window
  sets the limit of the Session history.
- [Memory](../patterns/memory.md): the knowledge that stays across Sessions. It
  adds to the history in one Session.
- The pattern [PromptLoading](../patterns/prompt-loading.md) puts knowledge into
  the system prompt of the Session.
- [Continuity](../patterns/continuity.md): the protocol that lets work continue
  across two or more Sessions.
- [Agent](agent.md): how a Session gets a frontend and an Environment.
