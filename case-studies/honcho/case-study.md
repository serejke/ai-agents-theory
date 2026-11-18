<a id="case-study-honcho--theory-building-as-a-latency-hierarchy"></a>

# Case Study: Honcho, a Peer Theory in Layers of Latency

A Memory layer for an agent must keep knowledge about each peer for a long time,
and this knowledge must help the agent. A peer is a person or an agent that
Honcho keeps facts about. The Memory layer must give this knowledge back on
request. To store the messages without changes is easy, because a database does
it. The important decisions are different:

- Which types of derived knowledge must the system keep?
- When must the system make each type?

The structure of Honcho comes from these decisions.

<a id="summary"></a>

Honcho is a loop that continuously builds a peer theory for each peer. In this
case study, the **peer theory** is the knowledge with a structure about one
peer. The peer theory has levels. Each level has a link to the level below it.
The workers of Honcho are this loop. Each worker operates at the latency that
agrees with its level.

<a id="the-conceptual-primitive"></a>

## The peer theory

Each peer has a peer theory that becomes larger with time. The peer theory is
not a list of facts. It is a model of the peer with a structure, and it has
levels of depth:

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

Each level has a `source_ids` link to the level below it. Thus each fact of the
peer theory has a link to its source. An agent can follow a fact down through
the levels to the messages.

<a id="why-latency-is-the-organizing-axis"></a>

## The cause of the different latencies

Each level of the peer theory must operate at a different latency. These facts
show the cause.

### Knowledge has depth

Some facts about a peer come from one message, for example "Alice asked about
goroutines". Honcho can see some facts only across many messages, for example
"Alice tends to drop a question when she's stuck rather than push through". Some
facts tell who the peer is, and they change slowly, for example "Alice is a
backend engineer who works on auth". These layers are not interchangeable. Each
layer stays on the layer below it.

### Deeper layers use more tokens

A fact from one message comes from one small LLM call on that message. A pattern
across messages comes from an LLM that reads many facts together and does
reasoning about them. This uses more tokens. The peer card comes from careful
work on top of the patterns. This uses more tokens again.

### Deeper layers change less frequently

Honcho must find the facts of each message when the message comes. New messages
are the only source of these facts. Honcho can see a pattern only after
sufficient new data collects. If Honcho calculates a pattern again at each
message, it gets the same result. The facts about who the peer is change more
slowly. If Honcho calculates them continuously, it uses tokens for no result.

### The result: layers of latency

These facts all give the same structure. The step with the minimum depth and the
smallest quantity of tokens must run most frequently. The step with the most
depth and the most tokens gets the smallest value from frequent runs. Two other
structures have problems:

- If a system makes each layer when a message comes, it uses too many tokens on
  patterns that did not change.
- If a system makes each layer only at query time, each reader does again the
  work of the previous reader.

The structure that prevents the two problems is **layers of latency**. Each
layer of the peer theory operates at its rate. The rate of a layer agrees with
the rate at which its input collects sufficient new data.

Thus latency does not only tell how Honcho runs. Latency divides the work,
because depth, tokens and the rate of change all change with it. The workers do
the same operation: they make a result at a higher level from the input that
collects. They operate at different rates, because their inputs collect at
different rates, and their outputs become expired at different rates.

<a id="workers-as-refinement-at-different-latencies"></a>

## The workers

Each worker of the loop operates at a different latency. The label of each
worker compares it to a step in how a person thinks.

- **Deriver: "perception".** The Deriver operates when messages come. It uses a
  small number of tokens and operates on batches. It makes the lowest level: it
  changes messages into `explicit` facts. It does one LLM call for each batch,
  with no tools and no steps of reasoning. Thus the path of each message uses a
  small number of tokens.
- **Dreamer: "reflection".** The Dreamer operates away from the path of the
  messages and uses more tokens. It runs only when no new messages come from the
  peer. It does the same operation as the Deriver: it makes a result at a higher
  level from its input. Its input is the set of facts. Its output is the
  `deductive` layer, the `inductive` layer and the peer card. The Dreamer first
  makes the `deductive` facts and then the `inductive` facts. Thus the second
  step sees the output of the first step. If new messages come, the Dreamer
  stops immediately, because the peer is active again.
- **Dialectic: "retrieval as inference".** The Dialectic runs during each query,
  and the query waits for it. It is not a worker, but it has the same function
  as the workers, at the latency of a query. It makes a reply to one query from
  the peer theory. Its reasoning can also add new `deductive` facts.
- **Reconciler: "housekeeping".** The Reconciler writes a copy of the internal
  data in external indexes and corrects the differences between them. It is a
  different type of worker. It operates on the storage of the peer theory, not
  on the peer theory.

<a id="what-the-alignment-produces"></a>

## The result

The shape of the peer theory follows the quantity of tokens of each step:

- The `explicit` layer is the largest. Its step uses a small number of tokens
  and runs at each message.
- The `deductive` and `inductive` layers are smaller. Deeper reasoning runs only
  after no new messages come from the peer for sufficient time. Then new
  conclusions are possible.
- The peer card is small and changes slowly. The step that uses the most tokens
  writes it.

This pattern is almost the same as a usual description of how a person keeps
knowledge: "encoding while awake, consolidation during sleep, lossy
summarization into long-term identity". This description only helps to show the
pattern. The important point is this: Honcho does the deeper reasoning only
after sufficient data collects. Thus the tokens that Honcho uses agree with the
value that it gets. The reasoning occurs at these times:

- When the agent reads, because then the agent has a query and the reply is
  important.
- When no new messages come from the peer, because then the latency has no
  limit.
