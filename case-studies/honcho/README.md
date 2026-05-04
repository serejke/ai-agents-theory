# Case Study: Honcho

You forward every message your agent and your user exchange to Honcho. Honcho quietly builds and maintains a profile of each participant — atomic facts, behavioral patterns, durable identity. When your agent later wants to know something about the user, it asks Honcho a plain-English question and gets back a grounded answer with provenance back to the source messages. No vector queries, no prompt engineering, no hand-curated memory schema.

Underneath, Honcho is a memory infrastructure for LLM agents that continuously refines a structured _theory of who each peer is_ and serves that theory back through a natural-language query endpoint. Background processes operate at different latencies to keep the theory current; what looks like a queue-driven service from outside is, conceptually, a continuous theory-building loop.

## What it looks like in practice

A small backend team shares an internal AI assistant for code review, debugging help, and on-call support. Three engineers use it daily: **Alice**, a senior who owns the auth service; **Bob**, six months in and still learning Go; and **Charlie**, the SRE who runs the deploys. Every message any of them exchanges with the assistant — and the messages they exchange with each other through it — is forwarded to Honcho.

Across sessions, ordinary things get said:

- Alice — _"just give me the snippet, skip the commentary"_, _"merging once CI's green, I'll watch the canary"_
- Bob — _"wait, why does this need a mutex?"_, _"what does Context cancellation actually do here?"_
- Charlie — _"p99 on auth blew SLO, who deployed in the last hour?"_, _"canary's clean, ship to prod"_

Two months later, Alice asks the assistant for help drafting an onboarding plan for Bob on the new payments service. Before generating anything, the assistant calls:

```python
alice.chat("What are Bob's strengths and gaps right now?", target="bob")
```

Honcho returns a grounded answer scoped to what _Alice_ has seen of Bob: _"Bob is solid on HTTP handlers and request validation, but his questions cluster around Go's concurrency model — channels, Context, mutex usage. He has paired with Alice on the auth service and his PRs there have merged with light revisions. He hasn't touched the payments codebase yet, so expect to ramp him on the storage and queueing patterns it uses."_

Three things to notice: nothing in that profile was hand-curated, the assistant didn't have to remember anything across sessions, and the recall happens in plain English rather than as a vector match. Every claim can be expanded back to the messages it came from.

### How the answer is built

The reply is synthesized from a layered profile Honcho builds in the background. Each layer links to the one below it:

| Level         | Example observation about Bob                                                                            |
| ------------- | -------------------------------------------------------------------------------------------------------- |
| **explicit**  | "Bob asked what Context cancellation does on March 5." — directly stated.                                |
| **deductive** | "Bob is actively learning Go's concurrency model." — logically necessary given the cluster of questions. |
| **inductive** | "Bob's knowledge gap is concentrated in concurrency, not in HTTP or storage." — pattern across sessions. |

A recall agent can call up an inductive claim and walk it down through the deductive observations that support it, then through the explicit facts those rest on, then through the actual messages. The profile is a provenance-tracked reasoning graph, not a flat list of facts.

### Different observer angles

The same Bob looks different from different observers:

- **Alice's view of Bob** is shaped by their pairing on the auth service — code-review back-and-forth, concurrency questions, the PRs she has approved. This is the view Alice's query above pulls from.
- **Charlie's view of Bob** is sparse: a single incident channel where Bob handled a rollback under pressure. Charlie's representation of Bob barely overlaps with Alice's.
- **Bob's view of himself** is built from his own messages — the things he asks, the things he says he's stuck on. Useful for an agent acting on Bob's behalf.
- **The omniscient view of Bob** is the union: everything Honcho knows about Bob across all observers and his own self-observation.

A query specifies the angle. `bob.chat("how does Alice usually critique my code?", target="alice")` asks for Bob's view of Alice through their interactions; `charlie.chat("when does Alice typically deploy auth?", target="alice")` asks for Charlie's view of Alice grounded in deploys he has watched. The same peer carries multiple representations, one per observer pair, and the assistant picks whichever angle fits the situation.

This case study is split:

- [**case-study.md**](case-study.md) — the conceptual core. Strip the orchestration machinery, what is Honcho actually doing? Levels of inference, worker stages as latencies, why the split is load-bearing.
- [**architecture.md**](architecture.md) — the structural picture. Data model, processing flow, design philosophy.

Read `case-study.md` first if you want the idea. Read `architecture.md` first if you want the substrate.

## External references

- [Honcho overview](https://docs.honcho.dev/v3/documentation/introduction/overview) — the official introduction to Honcho as an open-source memory library for stateful agents.
- [Honcho architecture](https://docs.honcho.dev/v3/documentation/core-concepts/architecture) — the official architecture page that `architecture.md` is built from.
