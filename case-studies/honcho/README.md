# Case Study: Honcho

**Honcho** is a [Memory](../../patterns/memory.md) service for agent programs.
It keeps facts about each person and each agent that uses an agent program. When
the agent sends a query in usual English, Honcho gives back a reply that comes
from these facts. The company Plastic Labs makes Honcho, and its source code is
public.

Honcho is a service that operates adjacent to the agent. The agent connects to
it through HTTP. Honcho has these parts:

- a FastAPI HTTP API
- a worker process that operates when no request waits for it
- a Postgres database.

The data model of Honcho has these items: workspaces, peers, Honcho sessions and
messages. A peer is a person or an agent. The agent sends messages to Honcho
with `POST` requests. To get facts back, the agent uses the chat endpoint of a
peer. This endpoint accepts a query in usual English. It gives a reply, and each
part of the reply comes from the stored facts.

The usual Memory structure keeps the full text of each conversation in a vector
store. Honcho does not keep an index of the messages.

Honcho stores **conclusions about each peer**. A conclusion is a fact that
Honcho derives from messages. Each conclusion has `source_ids` links to the
messages or to the previous conclusions that it comes from. To get facts back,
an LLM call with tools goes through this graph of conclusions. It does not
examine a flat vector store. [`case-study.md`](case-study.md) gives the concept
below this structure.

## The documents of this case study

- [`case-study.md`](case-study.md): the concept of Honcho without the system
  that operates it. It gives the levels of conclusions, the workers as different
  latencies, and the cause of this division.
- [`architecture.md`](architecture.md): the structure of Honcho. It gives the
  data model, the flow of the data and the rules of the structure.

For the concept, read `case-study.md` first. For the system below the concept,
read `architecture.md` first.

<a id="what-it-looks-like-in-practice"></a>

## An example

A small group of backend engineers uses one internal AI assistant. They use it
to examine code, to debug, and to find the cause of incidents. These engineers
use it frequently:

- **Alice** is the engineer with the most knowledge in the group. Alice controls
  the authentication service.
- **Bob** came to the group 6 months before this example. Go is a new language
  for Bob.
- **Charlie** keeps the services in operation. Charlie deploys the code.

The assistant sends each message to Honcho. These are the messages between an
engineer and the assistant. They are also the messages between two engineers
through the assistant.

Across many Honcho sessions, the engineers write usual messages:

- **Alice**: _"just give me the snippet, skip the commentary"_, _"merging once
  CI's green, I'll watch the canary"_
- **Bob**: _"wait, why does this need a mutex?"_, _"what does Context
  cancellation actually do here?"_
- **Charlie**: _"p99 on auth blew SLO, who deployed in the last hour?"_,
  _"canary's clean, ship to prod"_

After 2 months, Alice tells the assistant to help with a plan for Bob. The plan
is for the start of the work of Bob on the new `payments` service. Before the
assistant writes a plan, it sends this query:

```python
alice.chat("What are Bob's strengths and gaps right now?", target="bob")
```

Honcho gives a reply. The reply uses only the facts about Bob that Alice saw:
_"Bob is solid on HTTP handlers and request validation, but his questions
cluster around Go's concurrency model — channels, Context, mutex usage. He has
paired with Alice on the auth service and his PRs there have merged with light
revisions. He hasn't touched the payments codebase yet, so expect to ramp him on
the storage and queueing patterns it uses."_

This example shows these properties of Honcho:

- No person wrote that profile manually.
- The assistant did not keep facts across Honcho sessions.
- The query is in usual English. Honcho does not use a vector store to find the
  facts.
- Each part of the reply has a link back to the messages that it comes from.

<a id="how-the-answer-is-built"></a>

### The layers of the reply

Honcho makes the reply from a profile with layers. Honcho builds this profile
while no request waits for it. Each layer has a link to the layer below it:

| Level       | Example conclusion about Bob                                                  | Source of the conclusion                            |
| ----------- | ----------------------------------------------------------------------------- | --------------------------------------------------- |
| `explicit`  | "Bob asked what Context cancellation does on March 5."                        | A message gives it directly.                        |
| `deductive` | "Bob is actively learning Go's concurrency model."                            | It must be correct because of the group of queries. |
| `inductive` | "Bob's knowledge gap is concentrated in concurrency, not in HTTP or storage." | It is a pattern across Honcho sessions.             |

An agent can start at an `inductive` conclusion and go down the layers. First,
the agent reads the `deductive` conclusions below it. Then it reads the
`explicit` conclusions below those. Then it reads the messages. Thus the profile
is a graph of reasoning, with a link from each fact to its source. It is not a
flat list of facts.

<a id="different-observer-angles"></a>

### Views from different observers

Each observer has a different view of the same peer. For Bob, Honcho keeps these
views:

- **The view of Alice.** Alice and Bob operated together on the authentication
  service. This view comes from the code inspections, the queries of Bob about
  parallel code, and the code changes that Alice accepted. The query of Alice in
  the example uses this view.
- **The view of Charlie.** This view has a small number of facts. It comes from
  one incident channel. In that incident, Bob quickly moved a service back to
  its previous version. This view is almost fully different from the view of
  Alice.
- **The view of Bob about Bob.** This view comes from the messages of Bob: the
  queries of Bob and the problems that Bob tells about. An agent that operates
  for Bob can use this view.
- **The full view.** This view holds all the views together: each fact about Bob
  from each observer and from Bob.

A query gives the view. For example:

- `bob.chat("how does Alice usually critique my code?", target="alice")` gets
  the view of Bob about Alice. It comes from the conversations between Bob and
  Alice.
- `charlie.chat("when does Alice typically deploy auth?", target="alice")` gets
  the view of Charlie about Alice. It comes from the times when Charlie saw
  Alice deploy code.

Thus one peer has two or more profiles, one for each observer. The assistant
selects the view that agrees with the task.

## External references

- ["Overview"](https://docs.honcho.dev/v3/documentation/introduction/overview):
  the first page of the documentation of Honcho. It tells that Honcho is a
  Memory library with public source code, for agents that keep their data across
  Sessions.
- ["Architecture"][architecture-page]: the page of the documentation of Honcho
  about its structure. `architecture.md` comes from this page.

[architecture-page]:
  https://docs.honcho.dev/v3/documentation/core-concepts/architecture
