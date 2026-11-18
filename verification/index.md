# Verification

Verification is the work that shows that an agent system operates correctly.
Usual tests are not sufficient, because the LLM in the system can give different
outputs for the same input. Verification is not a tier of the theory. It is a
set of methods that applies to each part of the system at its boundaries.

<a id="why-it-matters"></a>

## Value

Usual software is deterministic. The same code with the same input gives the
same output. A test calls a function and examines the result. If the test passes
one time, it passes each time.

An agent system does not obey this rule:

- The [LLM](../primitives/llm.md) at the core of the system is not
  deterministic. The same prompt can give different reasoning, different
  sequences of tool calls and different outputs.
- The [AgentLoop](../harness/agent-loop.md) makes this effect larger. The LLM
  reads each tool result and selects the next step from it. Thus a small
  difference at the start of a run can give a very different trajectory.

You can examine an agent system with tests. An agent system has many layers, and
most of these layers are deterministic. The core that is not deterministic must
have different verification methods from the deterministic shell around it. Know
the layer that you examine and the methods for that layer. Then you are sure
that the system operates correctly, and you do not only think it.

<a id="the-shape-of-verification"></a>

## The parts of verification

Verification has these parts:

- **[Trace](trace.md)**: the data that you record. A trace is the sequence of
  the LLM calls, the tool calls and the results between them in one run of an
  [AgentLoop](../harness/agent-loop.md). Each verification method examines a
  part of the trace.
- **[Seams](seams.md)**: the points where a test replaces a part. A seam is at
  each point where two primitives connect. At a seam, a test can put a different
  implementation, for example a mock LLM or a channel with prepared data. It can
  also put a tool that gives recorded results. Seams let a test examine an agent
  system.
- **[Methods](strategies.md)**: the checks that you do, and the time when you do
  them. Each method uses traces and seams differently. The methods are shell
  tests, property checks, LLM judges, trajectory checks and snapshot tests.

These parts are layers. The traces give the data that a test can see. The seams
give the points in that data where a test can replace a part. The methods tell
the checks to do at each seam and how frequently to do them.

## The verification stack

The [verification methods](strategies.md) make a stack. At the top are the
methods that are free and deterministic. At the bottom are the methods that use
the LLM, and are not deterministic. The model provider charges for each LLM
call.

```
                          Cost    Determinism   Frequency
                          ────    ───────────   ─────────
┌─────────────────────┐
│  Shell tests         │  Free    Deterministic  Every commit
│  (mock LLM)          │
├─────────────────────┤
│  Prompt snapshots    │  Free    Deterministic  Every commit
│  (golden files)      │
├─────────────────────┤
│  Property checks     │  $$      Non-det        On prompt/model change
│  (real LLM)          │
├─────────────────────┤
│  Trajectory checks   │  $$      Non-det        On agent logic change
│  (real LLM + trace)  │
├─────────────────────┤
│  LLM judge           │  $$$     Non-det        Scheduled (nightly/weekly)
│  (eval suite)        │
└─────────────────────┘
```

Each layer finds different problems. One layer is not sufficient:

- Shell tests find bugs in the connections between the parts. They do not find
  problems in the quality of the prompt.
- Property checks find errors in the structure of the output. They do not find
  small changes that make the quality worse.
- An LLM judge finds problems in quality. But it uses many LLM calls, and its
  result is not deterministic.
- Trajectory checks find errors in the steps of the agent. But they are possible
  only with a system that records traces.

An agent system for customers must have at minimum the shell tests and the
snapshots of the prompt. A better system also has property checks. The best
systems have the full stack.

<a id="design-implications"></a>

## Decisions for builders

**Agent builders**: put the LLM [seam](seams.md) in the code from the start.
Some agent code calls the LLM directly, for example with a command in the code
or with an inline API client. Then each test must call the LLM, or change the
code at runtime. If the LLM is behind a protocol, a test of the deterministic
shell is easy. For tests, no other decision about the structure of the code has
a larger effect.

**Pipeline builders**: record a [trace](trace.md) of each run. With traces, you
can debug errors, do a run again and add trajectory checks at a different time.
Record the traces also if you do not write trajectory checks. A trace is easy to
record during the run. After the run, it is hard to make again.

**Builders of quality tests**: use the [Evaluator](../patterns/evaluator.md)
pattern again. An agent can correct its work at runtime, with a retry loop and
an LLM judge. Then put the quality rules of that judge into one definition for
the agent and the tests. Use the same rules after the run to monitor the
quality. The agent and the quality tests then use one set of rules in two modes.

**The theory**: verification connects [Guardrails](../harness/guardrail.md) with
the quality checks after the run. A Guardrail prevents errors at runtime. The
two examine the same operations of the agent: tool calls, outputs and
trajectories. But they operate at different times.

A full agent system has the two layers. Guardrails stop dangerous operations
during the run. Verification shows correct operation after the run.

## Related

- [The trace](trace.md): the data that one run of an AgentLoop gives.
- [Seams](seams.md): the points at the boundaries of primitives where a test can
  replace a part.
- [Verification methods](strategies.md): the checks and when to run them.
- [Guardrail](../harness/guardrail.md): the runtime layer that prevents errors.
  Verification after the run adds to it.
- [Evaluator](../patterns/evaluator.md): the pattern that the LLM judge uses
  again.
- [AgentLoop](../harness/agent-loop.md): the harness that makes the traces that
  verification examines.
- [Channel](../patterns/channel.md): the boundaries of a channel are usual seams
  for verification.
