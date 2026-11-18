# Continuity

Continuity lets an agent continue a long task across two or more Sessions. Each
new Session starts at the point where the last Session stopped, and does not
start from zero. With Continuity, an agent can do a project of 30 days, not only
a task of 30 minutes. **Continuity** is a pattern: an agent system can use it,
but it is not necessary.

**Composition**: [Memory](memory.md) (ground truth + progress log) +
[PromptLoading](prompt-loading.md) (start sequence) +
[Guardrail](../harness/guardrail.md) (clean state) +
[Tool](../primitives/tool.md) (bootstrap).

<a id="why-it-matters"></a>

## Value

A [Session](../harness/session.md) can hold only the text that the context
window of the [LLM](../primitives/llm.md) can hold. Most projects are too large
for one context window. Thus the agent must use two or more Sessions. Without
Continuity, each new Session has these problems:

- **The agent stops before the task is done.** The agent sees the code and
  thinks that the task is done. In a new context window, a feature that is not
  fully done looks the same as a feature that is done. The agent can see the
  difference only if a document tells when the task is done.
- **The agent uses its context to find the condition of the project again.** It
  finds which work is done, which changes are open and how to start the
  Environment. This work is the same in each Session, and it does not move the
  task forward. Without a progress log and a bootstrap, each Session does this
  work again.

These problems have one cause. The agent has no data about the condition of the
project that stays after the end of the context window.

<a id="formal-definition"></a>

## Definition

```typescript
type Continuity = {
  groundTruth: GroundTruth; // what "done" means; stays across all Sessions
  progressLog: ProgressLog; // what is done and what is next; updated each Session
  bootstrap: Tool; // prepares the Environment, the same way each time
  startupSequence: PromptLoading; // the steps at the start of each Session
  cleanState: Guardrail; // the checks at the end of each Session
};
```

<a id="components-and-failure-modes"></a>

## Parts

Each part of Continuity stops one problem. If you remove a part, its problem
occurs again.

| Part           | Problem that it stops                                     |
| -------------- | --------------------------------------------------------- |
| Ground truth   | The agent stops before the task is done                   |
| Progress log   | The agent finds the condition of the project again        |
| Bootstrap      | The agent finds how to start the Environment again        |
| Start sequence | The agent starts new work before it knows the condition   |
| Clean state    | The agent gives open problems to the next Session to find |

<a id="ground-truth--prevents-premature-completion"></a>

### Ground truth

The ground truth is a list of the conditions that make the project done. Each
item tells one function that a user can see, and a test can examine it from the
start to the end. Each item also has a `status` field. The agent changes this
field only after this test.

```typescript
type GroundTruth = {
  items: Array<{
    description: string; // "user can open a new chat, type a query, see a response"
    steps: string[]; // the steps of the end-to-end test
    status: "done" | "todo"; // changed only after the end-to-end test
  }>;
  format: "json"; // a rigid structure that the agent does not change casually
};
```

The ground truth is a JSON file, not a Markdown file. The rigid structure of
JSON makes the agent careful. The agent sees the file as data that it changes
carefully, not as text that it can write again. Thus the format of the file
helps the agent to use it correctly.

Without a ground truth, the agent uses the code to find if the task is done.
This method can give an incorrect result. Code can be in the project but not
operate. A function can operate but not be fully done. The ground truth tells
clearly which work is done.

<a id="progress-log--prevents-archaeological-waste"></a>

### Progress log

The progress log is a file where the agent adds an entry at the end of each
Session. The agent does not change the previous entries. Each entry tells the
work that the agent did and the work that it completed. It also tells the
condition of the project at the end of the Session.

```typescript
type ProgressLog = {
  entries: Array<{
    session: string; // the identifier of the Session
    worked_on: string; // the work that the agent tried
    completed: string[]; // the work that the agent completed
    state: string; // the condition of the project
    next: string; // the recommended next step
  }>;
};
```

The progress log and the git history let each new Session find its position
quickly. A new Session reads the last entry. Then it knows the last work, the
next step and the condition of the Environment. It is not necessary to examine
the project to find these facts.

<a id="bootstrap--prevents-setup-tax"></a>

### Bootstrap

The bootstrap is a script, a tool or a procedure that prepares the Environment
for work in one step. It starts the servers, prepares the databases and installs
the dependencies.

```typescript
type Bootstrap = Tool & {
  name: "bootstrap";
  // Starts servers, prepares databases, installs dependencies.
  // The same in each Session: the agent makes no decisions.
  execute: () => Promise<{ output: "environment ready" }>;
};
```

After the first Session, each Session can start with the bootstrap. The agent
does not use tokens to find how to start the servers, prepare the databases and
prepare the program for a test. Each Session keeps these tokens. In a long
project, the total quantity of tokens that the agent keeps becomes large.

<a id="startup-sequence--prevents-disoriented-work"></a>

### Start sequence

The start sequence is a list of steps that each Session does before it starts
new work. The steps are the same each time. They collect the context that the
agent uses to find its position.

```typescript
const startupSequence: PromptLoading = () => {
  // 1. Read the progress log: the last work
  // 2. Read the ground truth: done work and open work
  // 3. Read the git log: recent changes
  // 4. Run the bootstrap: start the environment
  // 5. Run a basic test: make sure that the environment operates
  // Only then: select the next task and start work
};
```

The start sequence is a [PromptLoading](prompt-loading.md). It puts the data of
the other parts of Continuity into the context. If the basic test shows that the
Environment is broken, the agent repairs it before it starts new work. Thus the
agent does not add new work on top of broken work, where the problems become
larger.

<a id="clean-state--prevents-messy-handoffs"></a>

### Clean state

The clean state is a [Guardrail](../harness/guardrail.md) that does checks at
the end of each Session. The agent cannot stop its work until all the checks
pass:

- Git has no changes without a commit.
- The progress log has a new entry.
- The build passes.

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

This rule is a Guardrail, not a request in the system prompt. If the LLM can
ignore a rule, the LLM will ignore it at some time. A Guardrail is part of the
structure, and the agent cannot ignore it. Thus the agent does not give these
problems to the next Session: features that are not fully done, broken tests and
changes without documentation.

Git commits have two functions here:

- A commit message with a good description is documentation.
- A commit is a point that the agent can go back to. If a change fails, the
  agent can go back to the last commit that operated correctly.

<a id="continuity-vs-memory"></a>

## Continuity and Memory

Continuity composes [Memory](memory.md) with other concepts for one task: work
on a project across Sessions. Memory gives the facts that the agent knows from
previous work. Continuity gives the position of the agent in a task of two or
more Sessions.

| Item                 | Memory                                      | Continuity                                  |
| -------------------- | ------------------------------------------- | ------------------------------------------- |
| Function             | Knowledge across Sessions                   | The position in a task across Sessions      |
| Data                 | Decisions, facts and selections of the user | Ground truth, progress log and Environment  |
| When the agent reads | When the data is related to the task        | At the start of each Session, in a sequence |
| When the agent adds  | When it gets facts from the work            | At the start and the end of each Session    |
| Without it           | The agent does not keep the user selections | The agent does not know its position        |

The two patterns keep data across Sessions, but they do different work. They
compose independently:

- An agent can have Memory without Continuity. It keeps the selections of the
  user, but it cannot follow a task across Sessions.
- An agent can have Continuity without Memory. It follows the task, but it does
  not keep the fact that you want the TypeScript option `strict`.

<a id="when-continuity-matters"></a>

## When to use Continuity

If one context window can hold a task, Continuity is not necessary. For a small
bug repair, a check of a code change or a short conversation, the
[Session](../harness/session.md) history is sufficient.

Continuity is necessary in these conditions:

- **The task is too large for one context window.** Examples are a new feature,
  the work of a specification, and each project that uses more than one Session.
- **Two or more agents work on the same project.** Each agent must know the work
  of all agents, not only its last work. All the agents use the ground truth and
  the progress log to find the work of the other agents.
- **The project continues for many days.** The Session history does not stay.
  Without Continuity, a Session has no connection to a Session that occurred
  some days before it.

## Related

- [Session](../harness/session.md): the history in one Session. Continuity
  connects the Sessions.
- [Memory](memory.md): knowledge across Sessions. Continuity is the special type
  of Memory for the position in a task.
- The [PromptLoading](prompt-loading.md) pattern: the start sequence is a
  PromptLoading.
- [Guardrail](../harness/guardrail.md): the clean state is a Guardrail.
- [Workspace](workspace.md): the ground truth and the progress log are files in
  the Workspace.
