# Case Study: Hermes Agent

Hermes (Nous Research) ships a single Python class as the runtime for a terminal CLI, a multi-platform messaging gateway, a webhook-and-cron scheduler, and an Agent Client Protocol server for IDEs. Every surface instantiates that class against a shared SQLite session database, a shared full-text-searchable history, and a shared plugin / memory / skills system. You install it with `pip install`, run it from a terminal or a phone, and it carries state across surfaces. The same class is also the policy used during reinforcement-learning post-training of Hermes-family models, so improvements at the harness layer compose with improvements at the model layer.

This case study maps Hermes onto the theory's vocabulary, then notes ways the realisation in Hermes deepens how particular concepts can be utilised — not as new entries in the taxonomy, but as concrete depth within concepts the theory already names.

## Hermes Vocabulary

The terms below are Hermes-specific and recur throughout the case study. Bolding them on first use makes the table and notes scannable.

- **AIAgent** — the single Python class that realises every trigger surface and is also the RL policy.
- **HERMES_HOME** — profile-scoped home directory holding sessions, skills, memory files, trajectories, and logs.
- **Profile** — named configuration scope; one **HERMES_HOME** per profile, the unit of isolation between concurrent **AIAgent** instances.
- **Trigger surfaces** — CLI, Gateway (with platform adapters: Telegram, Discord, Slack, …), Cron + Webhook, ACP (Agent Client Protocol, for IDEs), and RL training. All instantiate **AIAgent** in-process.
- **SOUL.md / MEMORY.md / USER.md** — bounded files loaded into the system prompt at session start: persona, agent-curated memory entries, user-model entries.
- **SKILL.md** — a directory with a frontmatter-headed markdown body plus optional supporting scripts; defines a reusable invocable procedure ("a skill").
- **Slash commands** — `/skills install`, `/memory edit`, …; mutating ones default to next-session activation, with `--now` opt-in.
- **SessionDB** — durable SQLite session record with full-text and trigram (CJK-friendly) search; chains across context-compression via a parent-session-id link.
- **Trajectories** — opt-in JSONL session records under **HERMES_HOME**.
- **Honcho** — one of the optional external memory providers; dialectic user modeling.
- **Atropos** — Nous Research's RL environments framework; hosts Hermes's policy rollouts and reward functions.
- **Tinker** — Thinking Machines Lab's managed post-training service; consumes scored trajectories to update model weights.
- Agent-callable tools touching durable state: `memory`, `skill_manage`, `delegate_task`, `session_search`, `todo`.

## Hermes Mapped to the Theory

| Tier        | Theory term       | How it is realised in Hermes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Primitive   | **LLM**           | **Provider-agnostic adapters.** A family of per-provider adapters — Anthropic, the Codex Responses API, OpenAI Chat Completions, Bedrock Converse, Gemini Native, Gemini CloudCode, plus a Nous-hosted endpoint — sit behind a uniform interface. A model swap is configuration, not code. The LLM is treated as a commodity.                                                                                                                                                                                                                                                           |
| Primitive   | **Tool**          | **Two-tier dispatch.** State-touching tools (`memory`, `skill_manage`, `delegate_task`, `session_search`, `todo`) are intercepted in-process at the **AIAgent** level so they can read and write durable state directly. Everything else flows through a registry-based dispatch with pre/post hooks. The split keeps state mutation close to the agent's bookkeeping while leaving the long tail uniform.                                                                                                                                                                              |
| Primitive   | **Prompt**        | **Stable system prompt + user-message injection.** The system prompt — persona (**SOUL.md**), project context auto-discovered from the working directory, frozen snapshots of **MEMORY.md** and **USER.md**, the tool index, the skill index — is built once per session and held byte-identical across every API call within that session. Per-turn dynamic context (memory prefetch results, plugin context, **slash-command** skill invocations) is appended only to the latest user message, on a per-call copy that never enters the canonical conversation history.               |
| Harness     | **AgentLoop**     | **Synchronous while-loop with shared budget.** A single while-loop with three orthogonal stop conditions: a maximum iteration count, an **iteration budget** that can be shared with subagents (a delegated task borrows from the parent's pool) or forked, and a one-call grace flag that gives the model a final chance after the budget is exhausted. A separate interrupt flag breaks the loop when a new user message arrives mid-run.                                                                                                                                             |
| Harness     | **Session**       | **Logical session vs. runtime session.** The **logical session** is the **SessionDB** transcript chained across compression by parent-session-id; compression terminates the logical session and starts a child while the runtime session continues. The **runtime session** is a live **AIAgent** instance keyed by `(platform, chat_id, thread_id)`, owning a message queue and inline control-command handling. The runtime session is the concurrency boundary; the logical session is the search-and-recall boundary.                                                              |
| Harness     | **Guardrail**     | **Layered and multi-site.** A command allowlist and approval flow in front of shell tools. A regex-and-invisible-unicode injection scanner on cron prompts before scheduling. Per-tool domain-specific check functions on registration. Plugin pre-tool-call hooks that can short-circuit a call. The prompt-cache integrity rule — no mid-conversation context mutation — functions as an architectural guardrail enforced by where injection sites are wired and by code review.                                                                                                      |
| Harness     | **Agent**         | **Single AIAgent class for every surface.** **AIAgent** composes the Frontend ↔ Session ↔ Environment seams via parameter passing, callbacks, and the in-process boundary. Production turn, RL rollout, background reviewer, **ACP** session, cron job, webhook handler, and delegated subagent are all instances of the same class with different parameters.                                                                                                                                                                                                                          |
| Environment | **Environment**   | **Pluggable execution backends.** Local OS, docker, ssh, daytona, modal, singularity — selected by configuration. The active backend is what the terminal tool invokes. In RL rollouts the same backend is shared with the **Atropos** reward function, so the verifier inspects the actual files the model wrote during the rollout rather than a separate filesystem.                                                                                                                                                                                                                 |
| Pattern     | **PromptLoading** | **Skills, persona, project context.** **SKILL.md** files are directories on disk: a frontmatter-headed markdown body plus optional supporting scripts and templates. Persona is loaded from **SOUL.md**; project context is auto-discovered from the working directory. Skill invocations are injected as user messages, not system-prompt extensions, so the system prompt's cache prefix stays stable. **Slash commands** that mutate prompt-relevant state default to **deferred invalidation** — the change applies to the next session, with `--now` opt-in.                       |
| Pattern     | **Memory**        | **Pluggable providers with a single-external rule.** Built-in (**MEMORY.md** / **USER.md**) plus at most one external provider — **Honcho** for dialectic user modeling, plus vector-recall and knowledge-graph alternatives. Each turn opens with a non-blocking prefetch keyed off the original user message; an end-of-turn sync fires durable writes asynchronously; the next turn's prefetch is queued in the background so it is warm by the time the user replies. Memorisation is decoupled from the foreground turn — see "Writer and responder as separate Agent runs" below. |
| Pattern     | **StateMachine**  | **Per-turn agent state and cron job machines.** The agent loop carries explicit per-turn state — iteration count, **iteration budget**, grace flag, interrupt flag, checkpoint marker. Cron jobs are file-stored state machines with pause / resume / trigger / update transitions, governed by a file lock so multiple gateway processes coexist safely.                                                                                                                                                                                                                               |
| Pattern     | **Channel**       | **Multi-channel Gateway.** The Gateway is a fan-in/fan-out across messaging platforms. Each platform is a channel adapter with its own rate-limit, edit-message cadence, and content-type negotiation. Streaming output is paced per channel; background-process notifications carry per-channel verbosity controls.                                                                                                                                                                                                                                                                    |
| Pattern     | **Planner**       | **Not first-class.** Delegation via `delegate_task` is delegation, not planning; the `todo` store is a bounded scratchpad, not a plan tree. The agent's plan is whatever its reasoning content produces.                                                                                                                                                                                                                                                                                                                                                                                |
| Pattern     | **Router**        | **Several focused routers, no global one.** Provider routing inside the LLM adapter layer; image-generation backend routing; memory-provider routing inside the MemoryManager.                                                                                                                                                                                                                                                                                                                                                                                                          |
| Pattern     | **Evaluator**     | **Three sites.** **Atropos** reward functions in RL training environments are evaluators in the strict sense. The **background reviewer** that curates memory and skill artefacts is an evaluator over the recent transcript. Integration with eval harnesses (terminal-bench, mini-SWE) is a bulk evaluator over a benchmark dataset.                                                                                                                                                                                                                                                  |
| Pattern     | **Workspace**     | **Working directory, per-skill scope, RL sandbox.** The working directory is a configured, agent-visible scope. Each skill receives its own `${HERMES_SKILL_DIR}` variable. RL rollouts run in a `task_id`-keyed sandbox shared with the reward function — the same workspace is read by the model and by the verifier.                                                                                                                                                                                                                                                                 |
| Pattern     | **Continuity**    | **Cross-session record, persona, user model, parent-chain.** Across sessions: the durable **SessionDB** with cross-session full-text recall, **SOUL.md**, **USER.md**, the parent-session-id chain across compression. Within sessions: ephemeral memory prefetch surfaces relevant prior content into the current turn without persisting back into the new transcript. The agent-curated memory mechanism — a separate sub-agent run that reviews the recent transcript every few turns and decides what to write — is the durable-write half of Continuity.                          |
| Pattern     | **Topology**      | **Single-process multi-instance.** One Python process holds many **AIAgent** instances concurrently, isolated by **profile** rather than by process. Subagent fan-out is in-process `delegate_task`. Cross-surface routing is achieved by every trigger surface (CLI, Gateway, Cron, **ACP**) sharing the same in-process **AIAgent** class against the same persistent state under **HERMES_HOME**. The Frontend ↔ Session ↔ Environment split is realised topologically — frontend on a phone, session on a small VPS, environment in whatever the terminal tool points at.           |

## Concepts in Depth

The theory covers what Hermes does conceptually; what Hermes adds is depth in how particular concepts can be realised. The notes below each anchor to a concept the theory already names and describe a way Hermes utilises it that future case studies and concept documents can draw on.

### Prompt cache integrity as a runtime invariant

_Concept: Prompt, with implications for Guardrail._

Hermes treats the LLM provider's prompt cache as a runtime invariant the architecture is shaped around, not as a downstream optimisation. The system prompt — persona (**SOUL.md**), frozen **MEMORY.md** and **USER.md** snapshots, tool index, skill index — is built once per session and held byte-identical until the next session boundary. Everything that needs to be dynamic per turn lands in a per-call copy of the latest user message rather than in the system prompt or in canonical conversation history. **Slash commands** that would mutate prompt-relevant state default to next-session activation. The trade-off is explicit: the agent forgoes hot-reload of system-level state in exchange for stable cache prefixes across long conversations. The Prompt primitive's injection sites are organised around this invariant; the Guardrail layer enforces it not as a per-call policy but as an architectural-shape constraint on where injection is allowed to land.

### Two views onto durable harness state

_Concept: Memory, with implications for Continuity._

Durable harness state — the bounded **MEMORY.md** and **USER.md** files, the **SKILL.md** directory — has two distinct readers. The agent at session start reads them via a frozen snapshot loaded into the system prompt. The agent mid-session reads them via the conversation-history trace of its own tool calls and tool results, which always carry the live state because each `memory` or `skill_manage` response includes the post-write entry list. The disk file is the persistence layer and the handoff to the next session. The three views are complementary rather than competing, and a context-compression event is the one trigger that re-hydrates the snapshot mid-session, because compression destroys the tool-result trace that would otherwise carry the live state. The Memory pattern, realised this way, is not a single read path but a layered one — boot, runtime, persistence — each with a defined refresh trigger.

### Writer and responder as separate Agent runs

_Concept: Memory, with implications for Evaluator and Continuity._

Hermes does not append "remember to save important facts" to the foreground system prompt. Every few turns, a separate run of the same **AIAgent** class is launched in a fresh session whose prompt is "review this transcript and decide what to persist," with the `memory` tool available. The writer and the responder are different runs with different prompts. The foreground stays focused on the user's task; the reviewer specialises in writes-back. The same shape extends to skill creation: a post-task reviewer can call `skill_manage` to synthesise a new **SKILL.md** from a completed transcript without polluting the user-facing turn or its system prompt. Memorisation is realised as an Evaluator-shaped sub-Agent, not as a system-prompt instruction.

### The Evaluator shares the rollout's Environment

_Concept: Environment, with implications for Evaluator._

In RL rollouts the **Atropos** reward function and the agent share the same `task_id`-keyed Environment instance. The verifier's test command runs against the actual files the model wrote during the rollout, not a re-hydrated copy. This collapses the train-versus-serve gap at the Environment tier the same way running the same **AIAgent** class as the policy collapses it at the harness tier. Environment instances are typed by task identity, not by run identity, so the rollout-and-verifier pair is the natural sharing unit.

### Configuration scope as an isolation axis

_Concept: Topology._

The unit of isolation between independent **AIAgent** instances is a **profile**, not a process. A profile sets **HERMES_HOME**; everything that reads that path automatically scopes to the active profile — sessions, skills, memory, trajectories, logs. Multiple **AIAgent** instances can run concurrently in the same Python process with non-overlapping profiles. The cost of single-process multi-instance is that plugins and memory providers must be re-entrant; the benefit is that tools, plugins, memory, and skills compose across surfaces by default — a skill installed for the CLI is automatically available to a cron job and to the **ACP** session. Topology, realised this way, treats configuration scope as an isolation axis distinct from process scope.

### One Agent class across every trigger surface

_Concept: Agent, with implications for Topology._

Every **trigger surface** — CLI, Gateway (with platforms), Cron, Webhook, **ACP**, RL training rollout — instantiates the same **AIAgent** class with different parameters. Cron does not spawn a subprocess; webhooks synthesise an internal message event; the **Atropos** RL environment runs an instance of the same loop. The seams between use-cases are parameter flags rather than subclasses. The packaging cost is a class with many constructor parameters; the payoff is that every harness improvement composes immediately into every surface, including the policy under **Tinker** post-training. The Agent concept's claim — that Frontend, Session, and Environment are seams composed by a single runnable unit — is realised here without any branching by use-case.

### Persisted record is canonical, not per-call

_Concept: Prompt, with implications for Continuity._

**Trajectory** saving is opt-in and locally scoped to **HERMES_HOME**. The persisted record is the canonical conversation history — the user's literal messages, the model's responses, the tool calls and results — without the per-turn ephemeral injections. Memory prefetch results, plugin context, and skill bodies live only in the per-call API copy and never enter the persisted **trajectory**. Replay of a saved trajectory is therefore deterministic with respect to the canonical inputs, with the prefetch rerun against the live memory state at replay time. The Prompt primitive's per-call injection sites and the Continuity pattern's persistence layer are kept on different sides of the same boundary by construction.

## Memory: Pluggable Providers

Hermes ships memory as a plug-in subsystem with a uniform interface and a single-external-provider rule. The built-in provider is always loaded — the **MEMORY.md** and **USER.md** files; at most one of the external providers (**Honcho**, Mem0, Supermemory, ByteRover, Hindsight, Holographic, OpenViking, RetainDB) can be enabled alongside it. The single-external rule is enforced at provider-load time and exists to prevent the combinatorial mess of multi-provider memory: which provider's context wins, how many duplicate writes happen, what relevance heuristics do when they conflict.

Each provider conforms to the same lifecycle, called by the **AIAgent** loop:

- **Prefetch.** `prefetch_all(query)` runs at the start of each turn, keyed off the user's original message. The result is a context string injected into the per-call user message — never the system prompt — preserving the cache prefix. The call is non-blocking; slow providers do not stall the turn.
- **Sync.** `sync_turn(user, assistant)` runs at the end of each turn and fires durable writes asynchronously (on a daemon thread for providers that POST to a remote service).
- **Warm-next.** `queue_prefetch_all` runs after sync and starts the next turn's prefetch in the background, so the result is warm by the time the user replies.
- **Mirror.** `on_memory_write` is invoked when the foreground agent writes via the `memory` tool, allowing external providers to mirror writes that originated against the built-in store.

External providers can also register their own tools into the agent's tool set, exposing provider-specific operations directly to the LLM. Provider configuration lives under **HERMES_HOME**, so a profile selects its memory stack along with everything else.

What counts as a memory provider varies widely:

- **File-based** — the built-in **MEMORY.md** / **USER.md**: bounded, atomic-write, system-prompt-injected at session start, agent-callable via the `memory` tool.
- **Vector-recall** — query-by-similarity over chunked turn transcripts; returns top-k relevant excerpts at prefetch time.
- **Knowledge-graph** — structured entity-relationship store; returns a typed projection of the user's prior facts.
- **LLM-as-memory** — **Honcho** runs a remote dialectic LLM call of configurable depth (1–3) over the peer representation, against a 3-second timeout the next turn joins to harvest whatever's ready. The peer representation itself is refreshed in the background every few turns.

The interface is wide enough to hold designs that are mechanically incompatible — file-based and LLM-as-memory occupy different points in the storage / latency / cost space — but narrow enough that the agent loop does not need to know which one is active. The single-external rule is what keeps that uniformity tractable: every provider has to be the only external opinion in the room, so the lifecycle does not have to arbitrate between competing recommendations.

## Self-Improvement: Runtime Skill Creation

Hermes lets the agent modify its own action space at runtime by writing **SKILL.md** files. A skill is a directory under **HERMES_HOME** with a frontmatter-headed markdown body and optional supporting scripts. Once on disk, it becomes an invocable procedure — a `/<skill-name>` slash command, a cron `--skill` reference, or a parameter to `delegate_task`. This is harness-tier self-modification: the model is unchanged, but what the agent can do has grown.

The agent-callable entry point is `skill_manage`, with actions `create`, `update`, `view`, `list`, `delete`. On `create` the manager validates the frontmatter, runs a security scan (skills are injected into prompts in future sessions and must not carry injection or exfiltration payloads), and atomically writes `<HERMES_HOME>/skills/<name>/SKILL.md`. The skill is now persistent across sessions and across surfaces — a skill written from a CLI session is available to a cron job and to an **ACP** session in the same profile.

The synthesis-versus-invocation split is structural:

- **Synthesis** is performed by the **background reviewer** sub-Agent — the same writer-vs-responder split the memory subsystem uses. After a complex task, the reviewer reads the recent transcript, decides whether the procedure is generic enough to be reusable, and calls `skill_manage(action='create', ...)` if so. The user-facing agent stays focused on the task; the reviewer specialises in writes-back and runs in a fresh sub-session with its own prompt.
- **Invocation** is performed by the foreground agent or by the user via slash command. By default, a freshly-written skill becomes available **on the next session** — the system-prompt skill index is reloaded at session start, not mid-conversation. The `--now` flag opts into immediate visibility for the rare case it is needed.

Skills are dynamic at invocation time, not just at synthesis time. Skill preprocessing substitutes `${HERMES_SKILL_DIR}` and `${HERMES_SESSION_ID}` and optionally executes inline-shell snippets, so a skill can list current files, embed `git status` output, or pull live data into its instructions when invoked.

The constraints worth naming:

- **The agent that writes the skill is not the agent that invokes it.** Synthesis runs in the reviewer sub-Agent; invocation runs in the foreground or in a future session. This is a deliberate separation that prevents a foreground agent from "using a skill it just made up" mid-task.
- **Deferred invalidation by default** keeps the system prompt's cache prefix stable across the synthesis turn; a freshly-written skill enters the system prompt only on the next session boundary.
- **Security scan and frontmatter validation** sit on the write path because skill bodies are injected into prompts in future sessions; an unscanned skill would be a stored prompt-injection vector.

Runtime skill creation is the harness-tier counterpart to a separate, slower self-improvement loop at the model tier: production trajectories can be replayed, scored by **Atropos** reward functions, and fed to **Tinker** to update model weights for the next Hermes release. The two loops sit on the same **AIAgent** class — skill creation modifies the action space without touching weights, RL post-training modifies the weights without changing the action space. Both use a reviewer-shaped Evaluator as the gate that decides which artefacts persist.

## Related

- [LangGraph case study](langgraph.md)
- [PI case study](pi.md)
