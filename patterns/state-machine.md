# StateMachine

A StateMachine divides the work of an agent into phases. Each phase gives the
agent a different set of tools, and rules tell when the agent can go to the next
phase. A person can give the approval for a change of phase. For example, a
coding agent first reads the code, then waits for the approval of a person, and
only then can write files.

**Composition**: [Guardrail](../harness/guardrail.md) + a variable that holds
the phase + rules for the change of phase. Each phase is a
[Session](../harness/session.md) configuration with its tools, its system prompt
and its Guardrail policies. The LLM cannot change it. At each tool call, the
Guardrail reads the variable and lets only the tools of the active phase
operate. Only an external input changes the phase. Examples are the approval of
a person, the result of an Evaluator and a timeout.

<a id="why-it-matters"></a>

## Value

A StateMachine changes one long conversation into a workflow with steps, checks
and points to go back to. Without a StateMachine, the agent has the same tools
and the same permissions from the start of the Session to its end.

A usual workflow to write code has these phases:

1. Read the code and make a plan. The agent can only read.
2. Wait for the approval of a person.
3. Write the code. The agent has full access.
4. Run the tests to make sure that the code operates correctly.
5. Make a pull request.

Each phase has different tools, different permissions and a different result
that shows the end of the phase. A change of phase can wait for the approval of
a person, for the result of an Evaluator or for a guard.

Without a StateMachine, you have two options:

- One agent with all the tools. This is dangerous, because the agent can write
  code before it knows the problem.
- Two or more agents that you start and control, one for each phase.

A StateMachine lets a system give a workflow of phases directly. It is the part
that changes an agent in a terminal into an agent that operates as a CI/CD
pipeline.

<a id="formal-definition"></a>

## Definition

```typescript
type Phase = "plan" | "approve" | "implement" | "verify" | "deliver";

// The Session configuration of one phase: the tools and the system prompt
// that apply while the StateMachine is in this phase.
type PhaseConfig = {
  tools: Tool[];
  system: string;
};

type Transition = {
  from: Phase;
  to: Phase;
  trigger: "auto" | "human_approval" | "evaluation_passed" | "timeout";
  // Optional guard: reads the state around the agent (progress log,
  // Evaluator result, user flags) and tells if the transition can occur.
  guard?: (state: Record<string, unknown>) => boolean;
};

type StateMachine = {
  phases: Record<Phase, PhaseConfig>;
  transitions: Transition[];
  currentPhase: Phase;
  advance: (trigger: string) => Phase; // move to the next phase
};
```

Each phase is a different [Session](../harness/session.md) configuration, with
different tools, a different system prompt and a different output. The
StateMachine moves the agent from one configuration to the next configuration
with the rules for the change of phase.

<a id="full-workflow-example"></a>

## Example of a full workflow

```typescript
const developmentWorkflow: StateMachine = {
  phases: {
    plan: {
      tools: [fileRead, grep, glob], // read-only
      system: "Explore the code, propose a plan",
      output: "github_comment", // the plan goes to an issue comment
    },
    approve: {
      tools: [], // the agent does not work
      trigger: "human_approval", // waits for a comment from a person
    },
    implement: {
      tools: [fileRead, fileWrite, bash], // full access
      system: "Implement according to plan",
      output: "git_branch",
    },
    verify: {
      tools: [bash, screenshot],
      system: "Run tests, check visuals",
      evaluator: visualEvaluator, // an Evaluator as part of the phase
      onFail: "implement", // go back to implementation
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

Known systems show different levels of this pattern:

- The plan mode of Claude Code is a small StateMachine with only two phases:
  first a plan, then the code.
- A run of a GitHub Action has no phases. It does its work one time, from the
  start to the end.
- A full workflow must have a graph of phases with branches, steps back and
  approvals of persons.

<a id="anti-pattern-prompt-enforced-orchestration"></a>

## Phase sequence in the prompt only

Some systems write the sequence of the phases only in the system prompt of the
primary agent, and not in code.

```
# lead_agent.txt (prompt excerpt)
STEP 2: SPAWN RESEARCHER SUBAGENTS (IN PARALLEL)
STEP 3: WAIT FOR RESEARCH COMPLETION
STEP 4: SPAWN DATA-ANALYST SUBAGENT
STEP 5: SPAWN REPORT-WRITER SUBAGENT
```

This is a StateMachine with no control. Each change of phase is automatic, and
no guard examines it. No code makes sure that the previous phase was correct.
Only the reasoning of the LLM keeps the phases in sequence.

These errors can occur, and no part of the system stops them:

- The model does not do a phase, because it thinks that the phase is not
  necessary. For example, it does not start the `DATA-ANALYST` subagent.
- A phase gives an empty or bad output, and the pipeline continues with it.
- The model changes the sequence of the phases.

### The LLM as the guard

The same problem occurs in a different shape in graph frameworks. An edge with a
condition looks as a rule in code, but an LLM call makes the decision.

```typescript
// Looks like routing in code, but the prompt controls it:
graph.add_conditional_edges("verify", (state) => llm("should we proceed?"));
```

This guard can give a different result for the same input. The structure does
not make sure that the LLM selects the correct edge. In a production system, the
condition of an edge must examine the data of the workflow. It must not get the
decision from the LLM.

### When to use it

A phase sequence in the prompt only is satisfactory for some systems:

- first versions of a system
- examples that you show to persons
- pipelines where small errors in the sequence are not important.

If the system must make sure of the sequence, add guards in code:

```typescript
{ from: "research", to: "analyze",
  guard: (ctx) => glob("files/research_notes/*.md").length >= 2,
  trigger: "auto" }
```

<a id="implementation-approaches-graph-vs-imperative"></a>

## Graph or code

There are two methods to make a StateMachine. The two methods give the same
result: phases, changes of phase and guards.

- **Graph**, for example LangGraph and Google ADK. You declare nodes and edges
  and compile them into a program that runs. You give the shape of the graph
  before the program runs. Each change of phase is an edge. A function on an
  edge selects the next node.
- **Code**, for example durable execution engines or usual code. You write usual
  code that records the active phase. The flow of the code makes the shape. Each
  change of phase is a function call or a change of a variable.

| Property             | Graph                                  | Code                          |
| -------------------- | -------------------------------------- | ----------------------------- |
| Visual graph         | Included: the code is the graph        | Other tools are necessary     |
| Analysis before run  | Possible: can find nodes with no path  | Hard: the shape can change    |
| Routing at run time  | Hard: special code is necessary        | Easy: write `if/else`         |
| Large decisions      | Must go into edge functions            | Usual code                    |
| Tools for developers | Good: visual tools to change and debug | Small number: it is only code |

Usually, a graph is better for the routing in an agent. The shape of that
routing does not change, and a visual graph helps. Code is better for the
control of a full system. Its shape can change, and code is more flexible.

<a id="checkpointing-and-durability"></a>

## Checkpoints

Each phase is a good point for a checkpoint. If the system stops because of a
crash in a workflow, it can continue from the last completed phase. It does not
start again from the start.

The quantity of work between two checkpoints is important:

- **A checkpoint for each phase.** A completed phase stays after a crash. The
  system does again the work of a phase that is not completed. That phase starts
  again from its start, thus each phase must have idempotency.
- **A checkpoint for each step.** Each unit of work in a phase stays after a
  crash. After a crash, the system must do a smaller quantity of work again, but
  the infrastructure is larger.

This quantity tells how much work the system does again after a crash. It also
tells which idempotency your code must give. Select the quantity from the time
of a second run:

- If a phase uses 5 seconds of LLM calls, a checkpoint for each phase is
  sufficient.
- If a phase uses 30 minutes of API calls and operations on data, a checkpoint
  for each step can be necessary.

<a id="human-in-the-loop-as-interruptresume"></a>

## Approval of a person

Each approval of a person in a StateMachine uses the same mechanism. The system
stops, keeps the data of the workflow, waits and continues.

```typescript
// The transition that waits for a person
{
  from: "plan",
  to: "implement",
  trigger: "human_approval",
  // Mechanism: save the current state, stop execution,
  // wait for an external signal, load the state, continue
}
```

When the system continues, it must select which work runs again:

- If all the phase runs again, as in some graph frameworks, the phase must have
  idempotency. Two calls to the same APIs must give the same result.
- If only the work after the point where the run stopped continues, the
  checkpoint must keep the accurate condition of the run.

The mechanism is always the same. Two decisions tell if the system operates
correctly: which work runs again, and which work must have idempotency.

<a id="the-cognitive--execution-layer-split"></a>

## Reasoning and operations

The phases of a StateMachine show two different layers of an agent system:

| Layer      | Function                                      | Hard problem                           |
| ---------- | --------------------------------------------- | -------------------------------------- |
| Reasoning  | LLM calls, tool selection, reasoning, routing | Outputs that change, prompt changes    |
| Operations | API calls, changes to a database, payments    | Correct results, retries, compensation |

Phases with only LLM reasoning and phases with external operations must have
different infrastructure. A plan and an analysis are examples of LLM reasoning.
A reservation, a payment and a deployment are examples of external operations.
The reasoning layer must have changes to prompts and checks of the outputs. The
operation layer must have retries, compensation and idempotency.

The layers tell which infrastructure a system must have:

- **Hard reasoning and easy operations**, for example chat assistants, research
  agents and coding agents. A framework of agents that controls the two layers
  is sufficient.
- **Easy reasoning and hard operations**, for example reservation agents and
  payment flows. A durable execution engine is sufficient.
- **Hard reasoning and hard operations.** The layers compose. The framework of
  agents controls the reasoning. Phases that do dangerous operations call a
  durable execution engine.

Make the protection of each step agree with the damage that an error of the step
can cause. If the LLM makes up a fact, one retry corrects it. If the system
makes the same payment two times, the company must give the money back. Also,
the trust of the customer decreases.

<a id="concern-bundling-vs-separation"></a>

## One library or different services

Some frameworks, for example [LangGraph](../case-studies/langgraph.md), put
these functions into one library:

- orchestration: the routing of the graph and the checkpoints
- the data of the workflow: the data that all nodes use, and its storage
- the work: the tool calls and the code of the product.

The other option is to keep them apart, with a different service for each of
these functions.

One library is good for systems with hard reasoning and easy operations. Some
systems have no dangerous operations that must have durable execution, for
example payments or external API calls that must have compensation. In these
systems, a different orchestration service only adds work. Different services
become necessary when the operations become harder. Examples are these:

- The system must schedule work across all its parts.
- The system must control the work of two or more services.
- The operations of the company must have durable execution.

The correct selection comes from the position of the system between hard
reasoning and hard operations. Each option is correct for some systems. If you
know which option you selected, you prevent problems when the system becomes
larger.

## Related

- [Session](../harness/session.md): one continuous conversation. A StateMachine
  connects two or more Sessions with different configurations.
- [Guardrail](../harness/guardrail.md): controls the operations in a phase. A
  StateMachine controls the changes between phases.
- [Evaluator](evaluator.md): its result frequently starts a change of phase
  (`evaluation_passed` or `evaluation_failed`).
- [Agent](../harness/agent.md): a StateMachine is an important part of
  production Agents.
- [LangGraph case study](../case-studies/langgraph.md): a framework with the
  StateMachine as its product.
