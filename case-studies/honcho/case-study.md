# Case Study: Honcho — Theory-Building as a Latency Hierarchy

Honcho's surface is a memory API. Strip the queues, locks, and Postgres machinery and what remains is a **cognitive architecture organized by latency**. The background workers are not separate "things to do" — they're the same operation (refine accumulated input into a higher-abstraction artifact) running at three different timescales.

## The conceptual primitive

Each peer in the system has a growing artifact: a **theory of who they are**. Not a list of facts — a structured model with multiple levels of inferential depth:

```
messages          (raw evidence)
  ↓
explicit facts    (atomic, directly stated)
  ↓
deductive facts   (logically necessary given the explicits)
  ↓
inductive facts   (patterns / regularities across multiple sources)
  ↓
peer card         (the curated self-image, capped and durable)
```

Every stage carries a `source_ids` link to the stage below it. The whole thing is a **provenance-tracked theory** — you can always ask "why do you believe this?" and walk down to ground truth. This is the bit that makes Honcho's memory feel less like a RAG index and more like a knowledge base that knows where it got its beliefs.

## Workers as refinement at different latencies

The cognitive loop has three roles, distinguished only by the timescale on which they operate:

- **Deriver = perception.** Online, cheap, batch-friendly. It only does the bottom rung: messages → explicit facts. One LLM call per batch, no tools, no reasoning. The commitment is _don't think hard at write time_.

- **Dreamer = reflection.** Offline, expensive, only when the peer is idle. The same operation as the deriver — refine input into a higher-abstraction artifact — but the input is now the fact base itself, and the output is the deductive/inductive layers and the peer card. Internally split into a deduction phase and an induction phase so the second sees the first's output; the split is sequencing, not a different mode of cognition. Cancelled the moment new messages arrive (the user is active again, defer).

- **Dialectic = retrieval-as-inference.** Not technically a worker, but conceptually it belongs in the same family — it's the per-query latency. It refines the theory into a one-off answer for a specific question, possibly creating new deductive facts along the way as a side effect of reasoning.

- **Reconciler = housekeeping.** Replicates internal state to external indexes, heals drift. Conceptually a different category — it operates on the substrate, not on the theory.

## Why the split is load-bearing, not arbitrary

The architectural commitment is that **inference depth scales inversely with frequency**. Most data lives at the explicit layer because explicit extraction is cheap and runs constantly. Deductive and inductive layers stay smaller because deep reasoning only runs on idle peers. The peer card is tiny (capped at a few dozen entries) because it's curated by the slowest, most expensive process.

This mirrors how human memory is often described — encoding while awake, consolidation during sleep, lossy summarization into long-term identity — but the meaningful claim isn't the biological analogy. The meaningful claim is **economic**: you can't afford to do deep reasoning on every message, and you don't have to, because most of what's worth concluding only becomes visible after enough evidence has piled up.

The split also reverses a common mistake in "memory" systems: trying to be smart on every write. Honcho's commitment is the opposite — write fast, read smart, sleep deeply. Reasoning is paid for at _read_ time (when the agent has a specific question and a specific cost-of-being-wrong) and at _idle_ time (when the user is gone and latency doesn't matter). Reasoning is never paid for at _write_ time, when the cost is multiplied by message volume.

The one-line summary: **Honcho is a continuous theory-building loop over each peer, where the theory has typed inference levels, every level carries provenance to the level below, and the workers are that loop running at the latency appropriate to each level.** The queue, locks, and Postgres are how you make the loop survive concurrency and crashes — they are not the idea.
