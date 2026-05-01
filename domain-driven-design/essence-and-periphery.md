# Domain-Driven Design in the LLM Era

Most senior architects starting a new system today face a strange first decision: for this domain — a tennis coach's CRM, an HR recruitment platform, a clinical workflow, a small-business operations tool — which facts belong in a relational schema, which belong in a vector store, and which belong nowhere because the LLM can hold them in conversational context? The instinct is to pick a side. Pick a side and the system will be wrong by next quarter, in a direction the side determined.

This document is about why the question itself is mis-framed, and what the right framing produces. It is foundational material adjacent to the rest of [this book](../README.md) — agents are runnable artifacts inside domain models, not domain models themselves — but the decomposition matters because every concrete agent inherits the boundaries the domain model draws. The audience is software architects who have shipped systems before and are now reflecting on what shifts when LLMs become first-class infrastructure.

The argument is sequential: name the wrong axis; replace it with the right one; show what survives of classical Domain-Driven Design and what extends; separate the irreducibly per-domain work from the mechanizable; produce a one-page artifact that drives everything downstream; close on what this means for the architect's job over the next decade.

---

## The Polarity That Misleads

The question reached for first is "should we use a database or let the LLM handle it?" That question implicitly takes a binary axis: structured persistence vs. model-mediated reasoning. Both options on this axis are wrong commitments, and both are wrong for the same reason — they ask the system to make a global decision about a heterogeneous reality.

**Pole 1: traditional structured design.** Every entity, every relationship, encoded in a normalized relational schema. Services wrap the schema with REST endpoints. Validation is at the service boundary. The LLM, if present, is a thin assistant that knows how to call the services. This is what most enterprise architecture has looked like for two decades. It works when the domain is well-understood and the data shape is stable. It fails when reality is fluid — when a parent's email contains five things at once, when a candidate's profile mixes resume facts with interviewer impressions and cultural-fit notes that fit no field, when "complaint" cannot reduce to an enum without losing the part that made it worth recording.

The Pole 1 failure mode is **forced reduction**. To make the email fit the schema, the system reduces it. The reduction loses the information. The system records `category: complaint` and discards the part the human cared about.

**Pole 2: LLM-first design.** No schema. The LLM reads, decides, stores notes in JSON, retrieves them when relevant. This is what most early agent prototypes look like. It works for exploratory workflows where the data is genuinely free-form. It fails when invariants matter — when the system needs to know which student a payment belongs to, when a refund must reference a specific invoice, when "send_invoice" must always reach a real recipient with a real amount.

The Pole 2 failure mode is **drift**. The LLM's representation of "Sasha" in March is not the same as its representation in September. There is no canonical record. There is no audit trail. Two queries about the same student return contradictory information, and no one can tell which was right.

Both poles are responses to a real constraint — Pole 1 to the need for invariants, Pole 2 to the need for fluidity. The mistake is treating these as system-wide commitments. Different facts in the same domain belong at different levels of structural commitment, and the architect's first job is to decide that level fact-by-fact.

A note on terminology before continuing: both poles store schemas. Pole 1 stores its schema in PostgreSQL where it is enforceable, inspectable, and brittle. Pole 2 stores its schema in the LLM's training and prompt-evoked behavior, where it drifts, cannot be inspected, and is unverifiable. Choosing between them is choosing where the schema lives, not whether one exists. The framing this document offers is: stop choosing globally, start choosing per fact.

---

## Tiers of Structural Commitment

Every business domain has facts at distinct levels of structure. Naming them clearly is the foundational move.

**L1 — Invariants.** Facts that must hold or the business is broken. They are the contractual reality of the system.

In the tennis coach's CRM:

- Every `Student` has at least one `parent_contact`.
- Every `Lesson` references exactly one `TimeSlot` and one or more `Students`.
- Every `Invoice` has an `amount`, a `payee`, and a `status ∈ {draft, sent, paid, overdue, void}`.
- Cancellations are soft-delete; the audit trail is immutable.

In an HR recruitment platform:

- Every `Candidate` is identified by a verified `email_address`; a duplicate email triggers a merge workflow, not an insert.
- Every `Interview` has a `panel` of one or more `Interviewers`, a `time_slot`, and a `stage`.
- Every `Offer` references a single `Candidate`, a single `Role`, and an `expires_at`.
- Status changes on a `Candidate` are append-only with the actor's identity.

L1 facts deserve hard schemas, hard types, hard database constraints. They live in PostgreSQL or its equivalent. They are validated at write. The LLM never decides whether they hold; it reads them and writes them through narrow, validated interfaces.

**L2 — Conventions.** Facts that are usually true but legitimately vary. They shape defaults and validation suggestions without rejecting deviation.

Tennis CRM:

- Lessons are 60 minutes (sometimes 30, sometimes 90).
- Parents pay monthly (sometimes per-lesson, sometimes by package).
- A student has one assigned coach (but can take occasional lessons with another).

HR recruitment:

- The pipeline runs `applied → screen → onsite → debrief → offer → close` (but a referral may skip to `screen`, an internal transfer may skip to `debrief`).
- Onsite duration is four hours (but custom panels run longer).
- Interviewers are matched to stage by competency tag (with override).

L2 facts deserve soft schemas: typed defaults, suggestion engines, validation warnings. The system flags deviations; it does not reject them. The LLM can suggest, the system can warn, the human can override. This tier is where most of the configuration UX lives in a mature product.

**L3 — Fabric.** Facts where imposing structure falsifies reality. They are narratives, observations, judgments.

Tennis CRM:

- "Why this parent prefers Anna over the previous coach."
- "What happened in Tuesday's lesson that left Sasha upset."
- "The tone of last week's complaint and Anna's read on whether it was about price or scheduling."

HR recruitment:

- "Why the panel rejected this candidate" — three interviewers each gave overlapping but non-identical reasons.
- "The cultural-fit observation from a thirty-minute coffee chat that fits no rubric."
- "How this candidate handled the surprise architecture question."

L3 facts live as text — embedded, retrievable, semantically searchable. Forcing them into an enum or a JSON shape destroys the information that made them worth recording. They are not unstructured because the team was lazy. They are unstructured because reality is.

The architect's first move on a new domain is **tier triage**: walking the domain's facts and deciding, for each, which level it belongs at. This is the irreducible work. It cannot be delegated to a tool, an LLM, or a framework. Wrong answers — putting `complaint.tone` at L1 with a four-value enum, or putting `student.payment_status` at L3 as free text — degrade the entire system from below.

---

## What Classical DDD Got Right, and What Extends

Domain-Driven Design has been right about the structural moves for two decades. Bounded contexts partition the system; aggregates own consistency; ubiquitous language anchors communication between business and engineering; domain events make state transitions observable; repositories abstract persistence. None of this was wrong.

What constrained classical DDD was an unstated assumption: every modeled fact had to live at L1. The persistence options available — relational databases, document stores with explicit schemas, event sourcing with typed events — all required structure up front. So practitioners did one of two things: forced L3 facts into rigid schemas (and lost information), or marked L3 facts as "TODO: extract someday" (and lost them anyway). The L3 layer existed in the business but had no architectural home.

LLMs change this by making L3 a first-class storage and retrieval tier. Embeddings, vector stores, semantic search, retrieval-augmented LLM calls — these are the L3 fabric pipeline. They give the architect a way to store unstructured observation without losing access to it. Anna's note about Sasha's bad Tuesday is no longer "lost" the moment it leaves the conversation; it can be ingested, embedded, retrieved when relevant, and surfaced to the LLM during the next interaction with that student's parents. The L3 fabric is no longer a place where information goes to die — it is infrastructure.

DDD does not dissolve. It extends downward. Bounded contexts still partition. Aggregates still own consistency — at L1. Ubiquitous language still anchors — across all tiers. Domain events still fire — when L1 transitions occur. What is new is that the L3 layer is a real place where domain knowledge accumulates without having to be reduced.

A consequence: the _ubiquitous language_ now spans tiers. The same word — "complaint" in the tennis CRM, "concern" in HR — has an L1 form (a record with id, status, timestamp), an L2 form (with conventional fields like severity that may or may not be populated), and an L3 form (the fabric of notes and observations attached to the record). The architect's job is to design the language so all three tiers refer to the same conceptual entity without forcing any one tier to carry the others' load.

---

## Essence and Periphery

With tiers in place, the architect's primary lens for a new domain becomes: what is essence (irreducibly per-domain) and what is periphery (mechanizable across domains)?

**Essence — designed, not generated.**

| Concern              | Why it's essence                                                                                                                                                                                                       |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tier triage          | Deciding L1/L2/L3 for each fact in this specific domain. Wrong answers degrade everything below.                                                                                                                       |
| State machines       | Lifecycles of each aggregate. Tennis: `prospect → trialing → enrolled → active → lapsed → churned`. HR: `applied → screen → onsite → debrief → offer → accepted/rejected`. Different in every domain, never derivable. |
| Action vocabulary    | Closed set of side-effecting operations the system can perform. `schedule_lesson`, `issue_invoice`, `send_offer`. The naming, parameters, and constraints are the design.                                              |
| Trust topology       | Who counts as trusted (the coach, the recruiter), who as user (a parent, a candidate), who as untrusted content (an inbound email, an inbound resume).                                                                 |
| Aggregate boundaries | Which facts must change together transactionally. Which can be eventually consistent. This is L1-tier design and survives unchanged from classical DDD.                                                                |
| Access predicates    | For each entity, who has standing to read or write it and on what terms. The data layer's contract.                                                                                                                    |

**Periphery — generated, not designed.**

| Layer                           | What's reusable across domains                                                                                                                                                                                                             |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| L1 enforcement                  | Schema → migrations → ORM types → REST endpoints → validators. Codegen from a spec. Tools like Prisma, Drizzle, sqlc, tRPC, zod.                                                                                                           |
| L2 soft-schema scaffolding      | Default-providing forms, suggestion engines, validation-warning UIs.                                                                                                                                                                       |
| L3 fabric pipeline              | Ingestion → chunking → embedding → vector storage → retrieval. Same machinery for every domain; the domain only configures what to ingest.                                                                                                 |
| Agent harness                   | [Session](../harness/session.md), [AgentLoop](../harness/agent-loop.md), [Guardrail](../harness/guardrail.md), [Topology](../patterns/topology.md), [Environment](../environment/environment.md). Universal across every agent ever built. |
| Tier glue                       | The tools that surface L1 records, L2 suggestions, and L3 retrieval into the agent's tool set. Pattern-level, not domain-level.                                                                                                            |
| Capability/authorization engine | The mechanism that enforces the trust topology. The _engine_ is reusable; the _policies_ are essence.                                                                                                                                      |

The line dividing these columns: **the engine is reusable, the configuration is essence.** Everything in the right column is generated _from_ the artifacts in the left. No tool can produce the left column. Tools can produce all of the right column from the left.

This is why naming similarity misleads so badly across domains. A tennis CRM has Students. An HR platform has Candidates. They sound similar; they have lifecycles, contact info, payment-or-compensation flows. So an architect proposes "let's build a generic CRM with AI." The proposal collapses within a quarter, because:

- A `Student`'s lifecycle is dominated by attendance and lesson cadence; a `Candidate`'s by hiring-funnel stages with compliance-mandated rejection logging.
- A `Student`'s tone-of-complaint matters and surfaces back to them; a `Candidate`'s never reaches them at all.
- `schedule_lesson` and `schedule_interview` look the same and are not — different parties, different consents, different reschedule semantics, different externally-visible obligations.
- The trust topology differs: a parent emailing about their own child has authority over their child's records; an interviewer messaging about a candidate has authority over their own observations of the candidate, not over the candidate's record.

Cross-domain reuse lives at the periphery. Cross-domain reuse at the essence layer fails, because essence is exactly the part where the domain's reality differs.

---

## The Action Vocabulary

Of the essence concerns, the action vocabulary deserves special attention because it sits at the intersection of domain modeling and security.

An action vocabulary enumerates every side-effecting operation the system can perform on behalf of any principal. Each entry is structured:

```typescript
type Action = {
  name: string; // e.g. "schedule_lesson"
  principal: PrincipalType[]; // who can authorize it
  parameters: ParamSpec[]; // each with type and trust requirement
  preconditions: Predicate[]; // what must hold before
  reversible: boolean | TimeWindow; // can it be undone, and within what window
  external_effects: ExternalEffect[]; // notifications, payments, irreversible actions
};
```

Worked entries from the tennis CRM:

```typescript
schedule_lesson: {
  principal: ["coach"],
  parameters: [
    { name: "student_id", type: "StudentId", trust: "CRM_TRUSTED" },
    { name: "time", type: "TimeSlot", trust: "validated" },
  ],
  preconditions: [
    "time slot is open",
    "student in {trialing, enrolled, active}",
  ],
  reversible: { window: "until 24h before time" },
  external_effects: ["notify parent by email"],
},

issue_invoice: {
  principal: ["coach", "system_scheduled"],
  parameters: [
    { name: "student_id", type: "StudentId", trust: "CRM_TRUSTED" },
    { name: "amount", type: "USD", trust: "computed_from_CRM" },
  ],
  preconditions: ["student has payment_card_on_file"],
  reversible: { window: "until paid; refund possible after" },
  external_effects: ["charge card", "email receipt"],
},
```

This artifact is doing two jobs simultaneously.

It is a **domain artifact** — it enumerates exactly what the business does. Reading the list tells you what the system is for. State transitions live here. Policy lives here. The dual entry of `principal` and `external_effects` makes the implications of every operation explicit, in one place.

It is also a **security artifact**. The architectural defenses with provable claims — capability-tagged plan execution, information-flow control, plan-then-execute (catalogued in [security/architecture.md](../security/architecture.md)) — all share an assumption: the agent's action space is closed and authored by a trusted principal. That assumption is not an overlay on top of the domain model. It _is_ the action vocabulary. A trusted principal authoring policy for a CaMeL-class system is doing DDD. The privileged planner LLM compiling that policy into a runnable program is mechanizing periphery from the action vocabulary spec.

This convergence is not coincidence. Domain modeling and agent security want the same thing: an enumeration, by a trusted human, of the bounded set of things the system is permitted to do. DDD calls it the action vocabulary. Security calls it the closed action space. The artifact is the same.

A consequence worth naming: when the action vocabulary is skipped — when teams rely on "the LLM will figure out the right action" or "the API endpoints implicitly define the actions" — they have given up both their domain model's coherence and their agent's safety guarantees in one stroke. The two failures look unrelated at the time; they are the same failure expressed in two registers.

---

## The Authorization Gap That Looks Like an Injection

The action vocabulary's `principal` field encodes who can call each action. That handles one half of authorization. The harder half is encoded in parameter trust: who can call the action _for which entity_. A worked failure mode makes this concrete.

In the tennis CRM, a parent named Maria emails parents@ asking when Vova plays next week. Vova is a real student. Maria is not Vova's parent. The agent's program — well-designed against prompt injection, structurally hardened along the lines of [the lethal trifecta defense](../security/first-principles.md) — answers Maria's question correctly: classify intent, look up student, return schedule, reply to sender. Every capability check passes. Maria has just learned when an unrelated child is at the courts.

This is not prompt injection. The email was honest. The agent did exactly what its program said to do. The bug is that the system had no concept of _who is allowed to ask about which student_.

Three threat classes belong in the architect's working vocabulary, and conflating them is the deeper bug behind every individual exploit.

| Class               | What's wrong                                         | Where it's enforceable                                                    |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------- |
| **Injection**       | Untrusted content overrides intended behavior        | Capability typing / dual-LLM / [Guardrail](../harness/guardrail.md)       |
| **Authorization**   | Legitimate request, but the requester isn't entitled | Tool contract / data layer access predicate                               |
| **Confused deputy** | Agent acts on behalf of A using B's authority        | Identity propagation through the [Tool](../primitives/tool.md) call chain |

The fix for the Maria case lives inside the domain's tool contracts, not inside the security overlay. Replace `lookup_student(name)` with `lookup_student_for_parent(parent_email, name)`, which returns the record only if the email is a registered parent contact for that student. The shape of the agent's program does not change; the data layer enforces the relationship at the boundary where the data lives. The capability system carries the parent's email through to the tool, but the _decision_ — that Maria is not Vova's parent — is encoded in the CRM, not in the capability system.

This is the access-predicate row in the essence table earlier. Access predicates are part of the domain model. They require knowing, for each entity, who has standing to read it and on what terms. No codegen produces them. They are designed.

The HR equivalent: an interviewer asking about candidates they did not interview. The action `read_candidate_observations` must be parent-scoped to "interviewers who participated in this candidate's panel," not "any interviewer in the org." Same shape, different domain language. Same essence work.

---

## Operational vs. Definitional Autonomy

A reasonable concern at this point: if the action vocabulary must be enumerated, doesn't that forbid autonomous agents that decide things on their own? The reframe is small but consequential.

The trusted principal's input does not have to be a one-shot command. It can be a standing policy:

> "Every morning, scan parents@ inbox. For schedule questions, reply with the child's lesson time. For cancellations, leave them in my review folder and notify me by SMS. For payment questions, forward to my accountant. For anything else, leave alone."

A privileged planner compiles this into a program with a `loop / sleep / classify / branch / dispatch` shape. Anna authored the policy once. The agent runs autonomously forever. The seed-once-run-autonomously mode is fully native to this design.

The constraint is not "must enumerate every workflow." It is: **the action space must be closed under what the trusted principal has authorized.** Two distinct things hide under the word "autonomy":

- **Operational autonomy** — running a loop forever, choosing within an authorized action space, picking which records to touch, deciding when to act. Compatible with everything in this document.
- **Definitional autonomy** — defining new action types or new policies in response to runtime content. Incompatible with this design, and arguably incompatible with any security model that distinguishes principals from attackers.

The reframe that makes this comfortable: **the planner LLM is a compiler from natural-language policy to enforceable code.** The trusted principal writes intent; the planner writes the implementation; the interpreter (or the executing agent harness) executes deterministically. The trust gradient runs from the principal through the planner to the runtime; values pulled from untrusted content fill program slots, they do not alter program structure. Autonomy is bounded by what the policy authorized — exactly the same arrangement every secure autonomous system has had for decades, from cron jobs to OS daemons to automated trading systems. The novelty is not "you cannot be autonomous"; it is "the policy must be written by a trusted principal, not implicitly by whatever the LLM decides on the fly."

For a coaching CRM or an HR platform this barely binds. Real customer-service workflows are well-bounded; the action types are mostly known in advance. For an "AGI personal assistant that figures out what to do with anything you throw at it," it is a hard ceiling.

---

## The One-Page Deliverable

The architect's first deliverable on a new domain is a single document. It fits on one page because it must — the moment it sprawls, the team has stopped designing essence and started prematurely writing periphery.

```
1. INVARIANTS (L1)
   5–15 facts that must always hold.
   Each phrased as a single declarative sentence.

   Tennis CRM:
     - Every Student has at least one parent_contact.
     - Every Invoice references a Student that exists.
     - Lesson cancellations are soft-delete; the audit trail is immutable.
     - A Student is in exactly one lifecycle state at a time:
       prospect | trialing | enrolled | active | lapsed | churned.

   HR recruitment:
     - Every Candidate is identified by a verified email; duplicate
       emails trigger merge, not insert.
     - Every Interview's panel has at least one Interviewer.
     - Status changes on a Candidate are append-only, timestamped
       with the actor's identity.

2. STATE MACHINES
   3–7 lifecycles, written as a transition table.
   For each transition: trigger action, principal, resulting state.

   Tennis CRM, Student lifecycle:
     prospect   --[trial_lesson_booked]-->        trialing
     trialing   --[enrollment_signed]-->          enrolled
     enrolled   --[first_lesson_attended]-->      active
     active     --[no_attendance_60d]-->          lapsed
     lapsed     --[reactivation_paid]-->          active
     lapsed     --[no_action_180d]-->             churned

3. ACTION VOCABULARY
   10–30 named operations, each tagged with:
     - principals authorized
     - parameters with types and trust requirements
     - preconditions
     - reversibility window
     - externally-visible effects

   Format:
     schedule_lesson:
       principals: coach
       params: student_id (CRM_TRUSTED), time (validated)
       preconditions: time slot open; student in {trialing, enrolled, active}
       reversible: until 24h before time
       external: notifies parent by email
```

That document is the essence of the system. It is the artifact a senior architect produces in the first week. It is reviewed with the business owner — Anna, the head of recruiting, the clinical lead — until both sides agree it accurately describes the business.

From this document, the entire periphery generates.

- L1 schemas come from the invariants and the state-machine state types.
- Migrations come from schema diffs.
- ORM types fall out of schemas.
- REST endpoints fall out of action signatures.
- Validators fall out of parameter trust requirements.
- Test fixtures cover the state-machine transitions exhaustively.
- The agent's tool set is the action vocabulary, with each entry's principal/trust requirements compiled into capability policies along the lines [security/architecture.md](../security/architecture.md) describes.

What this document does _not_ generate: itself. The architect writes it. The business owner validates it. No LLM produces it from a domain description, because the document _is_ the domain description in canonical form, and producing it requires deciding the L1/L2/L3 tier of every fact — the irreducible essence work.

---

## The Architect's Workflow

When a senior architect starts a new domain, the procedure is sequential. Each step produces an artifact the next step consumes.

**Step 1 — Conversational discovery.** A day with the business owner. Listen for the words they use repeatedly — those are candidates for the ubiquitous language. Listen for "always" and "never" — those are L1 invariant candidates. Listen for "usually" and "sometimes" — those are L2 conventions. Listen for the stories that get retold, the "remember when that parent..." anecdotes — those are L3 fabric, and the retelling is evidence the L3 layer matters.

**Step 2 — Tier triage.** Take every noun and verb from Step 1. Decide its tier. Be ruthless about pushing things down. Default to L3 for any fact whose structure is uncertain; promote to L2 when defaults emerge; promote to L1 only when violation would break the business. Most teams over-promote, and over-promotion is the most common source of system rigidity.

**Step 3 — Aggregates and state machines.** Group L1 facts into aggregates; the aggregate root owns transactional consistency for its members. Then for each aggregate, write the lifecycle. Force every transition to be triggered by a named action with a named principal. If you cannot name the trigger, the transition is implicit and dangerous — keep working until every transition has a name.

**Step 4 — Action vocabulary and trust topology.** Enumerate every operation. Tag each with principals, parameters, preconditions, reversibility, external effects. Identify principals (the trusted authors), users (subjects of authorization), and untrusted content (anything ingested from outside). Annotate every action's parameters with trust requirements: which must come from validated lookup, which can come from user input, which can come from untrusted content (almost none). This step will take longer than the rest combined; it is where most of the domain's complexity actually lives.

**Step 5 — Periphery generation.** Run codegen pipelines from the artifacts produced above. Schemas, migrations, types, services, agent tool definitions, capability policies. Most of this should be hours, not weeks. If it takes weeks, the periphery layer is wrong, not the domain artifact.

**Step 6 — Agent and L3 wiring.** Wire the agent's harness — [Session](../harness/session.md), [AgentLoop](../harness/agent-loop.md), [Guardrails](../harness/guardrail.md), [Topology](../patterns/topology.md) — to the generated tool set and capability policies. Stand up the embedding pipeline. Identify L3 ingestion sources — coach notes, parent emails, interviewer write-ups. Wire retrieval into the agent's prompt assembly so that L3 context is available where it is relevant.

**Step 7 — Iterate the one-pager, not the code.** When the business changes — and it will — change the one-page document first. Verify the change with the business owner. Then regenerate periphery. Code that drifts from the one-pager is a bug to be reverted, not a fact to be incorporated.

The discipline this procedure enforces is that the one-page document is the source of truth. The system's behavior is derived from it. Drift between them is detectable, because it shows up as code that does not trace back to a line in the document.

---

## Why Now, and What This Means for the Future of Software

For two decades, software architecture has been dominated by the cost of periphery. Microservices, REST conventions, ORM tooling, schema migration frameworks, deployment pipelines, observability stacks — all of this is infrastructure for moving plumbing from "expensive bespoke" to "cheap commodity." The economics rewarded teams that mastered periphery. The architects who shipped fastest were the ones who had solved the periphery problem before.

That phase has ended. Codegen, LLM-mediated UI, agent harnesses, retrieval pipelines, and modern type systems have collapsed periphery cost faster than any prior generation of tooling. A solo developer in a serious type-checked stack generates the L1 schema, types, services, validators, and admin UI for a midsize domain in an afternoon. The L3 fabric pipeline is a vendor-supplied product. The agent harness is reusable infrastructure. The marginal cost of well-shaped periphery is approaching zero.

Essence cost has not changed. Deciding the right tier for `complaint.tone`, the right state transition out of `trialing`, the right authorization rule for "Maria asks about Vova" — these took the same effort a decade ago as they take today. They are bottlenecked on understanding the domain, not on producing the artifacts. No tool will reduce this cost soon, because the cost is not in the artifacts; it is in the conversation with the business owner that produces them.

The economics flipped. Periphery used to dominate the cost of building software. Essence now dominates the value. Teams that understand this redirect their senior architects upward, toward domain modeling, trust topology, and action vocabulary. Teams that do not continue assigning their best architects to periphery problems that codegen has already solved.

The same decomposition shows up in three places that look unrelated until they are not.

- In domain modeling: essence is the L1/L2/L3 tier triage and the action vocabulary; periphery is everything generated from them.
- In [agent design](../README.md): essence is the trust topology, the tool set, and the principal-authored policies; periphery is the harness — Session, AgentLoop, Guardrail, Topology, Environment — which is reusable across every agent ever built.
- In [security](../security/first-principles.md): essence is the closed action space and the capability rules a trusted principal authors; periphery is the enforcement engine that mechanically applies them.

The unifying claim, distilled: **enumerate, by a trusted human, the closed set of things this system is permitted to do.** That enumeration is the artifact every other concern depends on. It is the work that does not get cheaper. It is the work the architect of the next decade is paid to do.

The future of software, for the senior architect, looks like this: less code written, more domain modeled. Less infrastructure tended, more business reality understood. Less plumbing, more essence. The tools to build everything around the essence have arrived. What is left is the part that was always the hardest, and is now the most leveraged.

---

## Related

- [Security: First Principles](../security/first-principles.md) — the lethal trifecta, agent-authority vs. process-authority, why architectural defenses subsume probabilistic ones.
- [Security: Architecture](../security/architecture.md) — sandbox tiers, the CaMeL-class capability execution defense, plan-then-execute patterns, the working checklist for a current agent design.
- [Tool](../primitives/tool.md) — where action contracts and access predicates live.
- [Guardrail](../harness/guardrail.md) — where capability rules from the action vocabulary are enforced at runtime.
- [Topology](../patterns/topology.md) — how the trust topology becomes a multi-agent shape.
- [Environment](../environment/environment.md) — where data-layer access enforcement physically lives.
- [Agent](../harness/agent.md) — the runnable artifact that the action vocabulary configures.
