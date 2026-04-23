# Verification

Verification is how you confirm that an agent system works. It is an orthogonal concern — not a primitive or pattern, but a set of strategies that intersect every primitive at its boundaries.

## Why It Matters

Traditional software is deterministic. Given the same input, the same code produces the same output. Testing is straightforward: call a function, assert on the result. If the test passes today, it passes tomorrow.

Agent systems break this contract. The core computation — the [LLM](../primitives/llm.md) — is non-deterministic. The same prompt may produce different reasoning, different tool call sequences, different outputs. The [AgentLoop](../harness/agent-loop.md) compounds this: each tool call produces a result that the LLM interprets to decide the next action, so small variations early in a run cascade into divergent trajectories.

The answer is not "you can't test agents." The answer is that agent systems have multiple layers, most of which are deterministic, and the non-deterministic core requires different verification strategies than the deterministic shell. Understanding which layer you're verifying — and what tools apply at each — is the difference between a system you trust and one you hope works.

---

## The Shape of Verification

Verification decomposes into three concerns:

- **[Trace](trace.md)** — what you observe. The ordered sequence of LLM calls, tool invocations, and intermediate results produced by an [AgentLoop](../harness/agent-loop.md). Every verification strategy operates on some subset of the trace.
- **[Seams](seams.md)** — where you substitute. Every primitive boundary is a point where one implementation can be swapped for another (a mock LLM, a pre-populated channel, a fake tool). This is what makes agent systems testable at all.
- **[Strategies](strategies.md)** — what you assert and when. Distinct techniques, each combining traces and seams differently: shell testing, property-based output verification, LLM-as-judge, trajectory verification, snapshot testing.

The three are layered: traces define the observable surface, seams define the substitution points on that surface, and strategies define what to check at each seam and how often.

---

## The Verification Stack

The five [strategies](strategies.md) compose into a stack, ordered by cost and determinism:

```
                          Cost    Determinism   Frequency
                          ────    ───────────   ─────────
┌─────────────────────┐
│  Shell testing       │  Free    Deterministic  Every commit
│  (mock LLM)          │
├─────────────────────┤
│  Prompt snapshots    │  Free    Deterministic  Every commit
│  (golden files)      │
├─────────────────────┤
│  Property assertions │  $$      Non-det        On prompt/model change
│  (real LLM)          │
├─────────────────────┤
│  Trajectory checks   │  $$      Non-det        On agent logic change
│  (real LLM + trace)  │
├─────────────────────┤
│  LLM-as-judge        │  $$$     Non-det        Scheduled (nightly/weekly)
│  (eval suite)        │
└─────────────────────┘
```

Every layer catches different failures. No single layer is sufficient:

- Shell tests catch plumbing bugs but miss prompt quality issues.
- Property assertions catch structural failures but miss subtle quality regression.
- LLM-as-judge catches quality issues but is expensive and non-deterministic.
- Trajectory checks catch behavioral issues but require trace infrastructure.

A production agent system needs at least the shell-test and prompt-snapshot layers. A serious one adds property assertions. A mature one has the full stack.

---

## Design Implications

**For agent builders**: Make the LLM [seam](seams.md) explicit from day one. If your agent code calls the LLM directly (hardcoded subprocess call, inline API client), every test requires a real LLM call or monkey-patching. If the LLM is behind a protocol, testing the deterministic shell is trivial. This is the single highest-leverage design decision for testability.

**For pipeline builders**: Capture [traces](trace.md). Even if you don't write trajectory assertions today, having traces means you can debug failures, replay runs, and add assertions later. A trace is cheap to capture and expensive to reconstruct after the fact.

**For eval builders**: Reuse the [Evaluator](../patterns/evaluator.md) pattern. If your agent already has runtime self-correction (retry loop with LLM-as-judge), extract the evaluation criteria into a shared definition. Run the same criteria offline for quality tracking. One set of criteria, two deployment modes.

**For the theory**: Verification connects [Guardrails](../harness/guardrail.md) (runtime prevention) with offline quality assurance. Both operate on the same action space — tool calls, outputs, trajectories — but at different points in the lifecycle. A complete agent system has both: guardrails that prevent catastrophic actions during execution, and verification that confirms correct behavior after execution.

---

## Related

- [Trace](trace.md) — the observable output of an AgentLoop
- [Seams](seams.md) — substitution points at primitive boundaries
- [Strategies](strategies.md) — the verification techniques
- [Guardrail](../harness/guardrail.md) — runtime prevention, complementary to offline verification
- [Evaluator](../patterns/evaluator.md) — the pattern reused for LLM-as-judge verification
- [AgentLoop](../harness/agent-loop.md) — what produces the traces that verification inspects
- [Channel](../patterns/channel.md) — channel boundaries are natural verification seams
