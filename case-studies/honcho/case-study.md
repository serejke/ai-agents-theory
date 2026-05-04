# Case Study: Honcho — Theory-Building as a Latency Hierarchy

A memory layer for an LLM agent has to maintain useful long-term knowledge about each participant and serve it back on demand. The interesting question isn't _how to store_ raw messages — that's a database — but _what kind of derived knowledge is worth holding_ and _when each kind should be produced_. The shape of Honcho falls out of taking those two questions seriously.

## Why latency is the organizing axis

**Knowledge has depth.** Some claims about a peer come from a single message ("Alice asked about goroutines"). Some become visible only across many messages ("Alice tends to drop a question when she's stuck rather than push through"). Some are durable identity facts that change slowly ("Alice is a backend engineer who works on auth"). These layers are not interchangeable: each one rests on the layer below it.

**Depth and cost rise together.** A single-message fact takes one cheap LLM pass over one message. A cross-message pattern requires reading many facts at once and reasoning about them, which costs more. A curated identity layer requires careful synthesis on top of patterns, which costs more again.

**Depth and the useful recompute rate move in opposite directions.** Single-message facts must be extracted as messages arrive — new messages are the only source. Patterns only become visible once enough new evidence has piled up to support them; recomputing them on every message returns the same answer. Identity is slower still; constant recompute is pure waste.

These three pull in the same architectural direction. The cheapest, shallowest inference is the one that needs to run most often; the most expensive, deepest inference gains the least from running often. A system that produces every layer on the write path overspends on patterns that haven't changed yet. A system that defers every layer to query time forces every reader to redo work the previous reader already did. The structure that resolves both pressures is a **latency hierarchy** — each layer of the artifact runs on its own clock, where the clock rate matches the rate at which the layer's inputs accumulate enough new signal to be worth re-reading.

Reading Honcho through a latency lens, then, isn't a deployment observation laid on top of the system. It's the axis the work factors along once depth, cost, and recompute rate are taken together. Several background processes run the same conceptual operation — refine accumulated input into a higher-abstraction artifact — at different timescales because their inputs accumulate at different rates and their outputs become stale at different rates.

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

Every stage carries a `source_ids` link to the stage below it. The whole thing is a **provenance-tracked theory** — any claim can be expanded down to the evidence it rests on, all the way to the actual messages.

## Workers as refinement at different latencies

The cognitive loop has roles distinguished by the timescale each operates on:

- **Deriver = perception.** Online, cheap, batch-friendly. It handles the bottom rung: messages → explicit facts. One LLM call per batch, no tools, no multi-step reasoning. The design choice is to keep the message path cheap.

- **Dreamer = reflection.** Offline, more expensive, runs only when the peer is idle. The same operation as the deriver — refine input into a higher-abstraction artifact — but the input is the fact base itself and the output is the deductive layer, the inductive layer, and the peer card. Internally split into a deduction phase and an induction phase so the second sees the first's output. Cancelled the moment new messages arrive: the peer is active again, defer.

- **Dialectic = retrieval-as-inference.** Runs synchronously on each query rather than as a background worker, but conceptually it belongs in the same family — it's the per-query latency. It refines the theory into a one-off answer for a specific question, sometimes creating new deductive facts along the way as a side effect of reasoning.

- **Reconciler = housekeeping.** Replicates internal state to external indexes and heals drift. A different conceptual category — it operates on the substrate that holds the theory, not on the theory itself.

## What the alignment produces

The shape of the artifact mirrors the cost structure:

- The **explicit** layer is the largest, because it's cheap and runs on every message.
- The **deductive** and **inductive** layers stay smaller, because deeper reasoning runs only when the peer has been idle long enough for new conclusions to be worth deriving.
- The **peer card** is small and slow-changing, because it's curated by the most expensive process.

The pattern parallels how human memory is often described — encoding while awake, consolidation during sleep, lossy summarization into long-term identity — though the analogy is descriptive rather than central. What's central is the **economic alignment**: deferring deeper reasoning until enough evidence has piled up keeps the cost curve aligned with the value curve. Reasoning happens at read time (when the agent has a specific question and the answer matters) and at idle time (when the latency budget is unconstrained).

## Summary

Honcho is a continuous theory-building loop over each peer. The theory has typed inference levels, every level carries provenance to the level below, and the workers are that loop running at the latency appropriate to each level.
