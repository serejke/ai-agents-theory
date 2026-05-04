# Case Study: Honcho

**Honcho** is an open-source memory service for LLM agent applications, authored by Plastic Labs. It runs as a FastAPI HTTP API plus a background worker process backed by Postgres — a standalone service next to an agent, integrated over HTTP. Its data model has four entities — **Workspaces**, **Peers**, **Sessions**, **Messages** — and a simple operational shape: messages go in via POST, conclusions come out through a per-peer chat endpoint that takes a natural-language query and returns a grounded answer.

What it replaces is the standard "raw transcripts plus a vector store" pattern. Instead of indexing messages, Honcho stores **derived conclusions about each peer**, linked via `source_ids` to the messages or earlier conclusions they came from. Recall is a tool-using LLM call over that provenance-tracked graph, not a flat vector lookup. The conceptual frame underneath is the subject of [case-study.md](case-study.md).

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
