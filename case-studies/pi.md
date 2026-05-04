# Case Study: PI

PI (the coding agent by Mario Zechner, distributed from the `pi-mono` monorepo) is the inverse of most coding agents in the market. Where Claude Code, Cursor, and Codex ship polished workflows with baked-in features, PI ships a minimal core and exposes every concept boundary as an extension point. The tagline on shittycodingagent.ai — "There are many coding agents, but this one is mine" — is not self-deprecation. It's an architectural stance: a coding harness should adapt to the user's workflow, not the other way around, and the way to achieve that is to refuse to decide which features belong inside.

PI is worth decomposing through the theory because it is, among the production coding agents, the one whose architecture is closest to the book's vocabulary. Its README explicitly argues "primitives, not features." Its extension system is organized around the exact concept boundaries the theory identifies — tools, hooks on tool calls, context injection, skill loading, session lifecycle. Studying it is studying what happens when someone designs a coding agent with roughly the same decomposition the book proposes, without having read the book.

---

## What PI Actually Is

The `pi-mono` monorepo ships seven packages; three of them constitute the agent framework. The rest (TUI, web-ui, Slack bot, GPU pod manager) are infrastructure.

| Package                         | Role                                                                                                                                                                                                                                                                      |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@mariozechner/pi-ai`           | Unified LLM provider layer (15+ providers, one `stream()` API). Implements the [LLM](../primitives/llm.md) primitive and the [Prompt](../primitives/prompt.md) primitive — a single structured message/tool format that gets serialized per provider protocol underneath. |
| `@mariozechner/pi-agent-core`   | Stateful [AgentLoop](../harness/agent-loop.md) with tool execution, event streaming, steering/follow-up queues, and `beforeToolCall`/`afterToolCall` hooks (the [Guardrail](../harness/guardrail.md) harness slot).                                                       |
| `@mariozechner/pi-coding-agent` | The `pi` CLI. [Agent](../harness/agent.md) wrapper (Frontend + [Session](../harness/session.md) + [Environment](../environment/environment.md)) with shared-prefix session-graph, extensions, skills, prompt templates, themes, and package manager.                      |

Collapsed to one line:

```
PI (the coding agent) = Sessions (AgentLoop + history, stored as a shared-prefix graph)
                     + Extensions (user-authored Tools, Guardrails, PromptLoading strategies, StateMachine phases, commands, UI)
                     + Skills (lazy PromptLoading, Agent Skills standard)
                     + PromptTemplates (eager PromptLoading, user-invoked)
                     + Compaction (lossy Session→Session transform)
                     + AGENTS.md/CLAUDE.md (eager PromptLoading, always-loaded)
                     + PiPackages (npm/git distribution of the four above)
                     + FourDeployments (interactive, print, JSON, RPC, SDK)
```

Every term after `Session` is a composition surface, not a built-in feature. This is the whole design.

---

## PI Mapped to the Theory

### LLM and Prompt — Provider Unification as a Library

`pi-ai` implements both the [LLM](../primitives/llm.md) and [Prompt](../primitives/prompt.md) primitives. It exposes a single structured message/tool format — effectively a provider-agnostic Prompt — that gets serialized per provider protocol underneath (Anthropic, OpenAI, Google, Azure, Bedrock, Mistral, Groq, Cerebras, xAI, OpenRouter, Vercel AI Gateway, and more). The LLM call that follows is stateless: `Prompt → text stream`. Subscriptions and API keys are both first-class auth modes. Thinking level is a uniform dial across providers that support it (`off | minimal | low | medium | high | xhigh`).

The theory claims LLM is a stateless function and the model is commodity, and that the Prompt is where provider-specific protocol lives. `pi-ai` operationalizes all three: the same `Agent` object binds to any of 15+ providers with no code change; the agent loop does not know or care which provider it is talking to; and the per-provider protocol serialization is hidden inside `pi-ai`'s Prompt abstraction. This is what "the model is commodity" looks like as a library boundary — and it only works because the Prompt layer is a real concept that `pi-ai` can unify across providers.

### AgentLoop — A While Loop with Extension Seams

`pi-agent-core` implements [AgentLoop](../harness/agent-loop.md) as a while loop, not a graph. The core cycle is identical to the theory's sketch:

```
while true:
    assemble Prompt from (system, messages, tool schemas, inserts)
    stream LLM response, parse tool calls from output text
    if no tool calls → return
    execute tool calls (parallel by default, sequential if any tool opts in)
    append tool results to Session history
```

The interesting part is the hook points the loop exposes — each corresponds to a theory concept boundary:

| Hook                                                             | Concept exposed                                                                          |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `transformContext(messages)` before each LLM call                | Prompt-assembly seam — where [PromptLoading](../patterns/prompt-loading.md) plugs in     |
| `beforeToolCall({ toolCall, args })` → `{ block, reason }`       | [Guardrail](../harness/guardrail.md) pre-action hook (structural tool-dispatch boundary) |
| `afterToolCall({ toolCall, result, isError })` → modified result | Guardrail post-action hook + audit                                                       |
| `convertToLlm(messages)` filter                                  | Message-type projection (custom agent messages invisible to the LLM)                     |

This is the book's claim — every concept boundary is a seam — rendered as a runtime API. Extensions subscribe to the lifecycle events (`tool_call`, `tool_result`, `context`, `before_provider_request`, `turn_start`, `turn_end`, `agent_start`, `agent_end`, `message_start`, `message_update`, `message_end`) and get to intercept at each seam.

### Session — Linear, but Sessions Share Prefixes

PI stores sessions as JSONL where each entry has `id` and `parentId`. A single file holds many branches; a pointer marks the active leaf. `/tree` moves the pointer. `/fork` creates a new session whose history begins at a chosen prior entry. `/clone` duplicates the active path.

Each leaf is still a linear [Session](../harness/session.md). `buildSessionContext()` walks from leaf to root and flattens, so the LLM always sees a linear history — unchanged from the theory's `Message[]` definition. What PI exposes is not a non-linear Session, but a navigable **graph of sessions** that share prefixes. Most systems treat sessions as isolated: fork means full copy, resume means load-one. PI makes the shared-ancestry relation first-class — sessions coexist in one file, branching is a pointer-aware write, and switching between them is a pointer move.

The refinement is Workspace-shaped, not Session-shaped. Session stays linear. What becomes navigable is a structured data space _over_ sessions — nodes are conversation states, edges are parent links. A workflow aimed at exploration ("try three approaches, keep the one that worked") benefits from exposing this graph; a conventional turn-by-turn chat does not need to.

### Tool — Bash + Read + Write + Edit (Minimal Set), Everything Else by Extension

The default tool set is four: `read`, `write`, `edit`, `bash`. Optional built-ins: `grep`, `find`, `ls`. That's it. No MCP, no sub-agents, no plan mode, no built-in todos. Every other capability is either:

- A **Skill** (markdown + scripts the agent invokes via `bash`), or
- An **Extension** (TypeScript that calls `pi.registerTool()`).

This is the theory's [Tool](../primitives/tool.md) primitive in its most unadorned form: tools are side-effectful functions; `bash` is a universal escape hatch; everything domain-specific composes on top. PI's rejection of MCP is consistent: MCP is one transport binding for tools, and if `bash` + markdown skills cover the same ground with lower operational cost, the binding is optional.

### Guardrail — The Tool-Dispatch Boundary

PI has no permission popup. The philosophy section says: "Run in a container, or build your own confirmation flow with extensions." But the machinery is always there: `beforeToolCall` and `afterToolCall` fire on every tool invocation. This is exactly the [Guardrail](../harness/guardrail.md) harness as the theory defines it — a structural interception layer at the tool-dispatch boundary, with the policy slot left empty by default. "PI has no guardrails out of the box" means "no policies registered," not "no interception point." The hooks are first-class API in `pi-agent-core`. The quick-start example literally blocks `rm -rf`:

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
    const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

This is a Guardrail policy registered at the dispatch hook. PI does not call it a Guardrail, but the type signature matches — and the hooks are the structural slot, whether or not policies fill them.

### PromptLoading — Laziness Tiers

The [PromptLoading](../patterns/prompt-loading.md) pattern shows up in several flavors, each at a different laziness:

| Mechanism                               | Laziness                  | When it fires                                                                                |
| --------------------------------------- | ------------------------- | -------------------------------------------------------------------------------------------- |
| `AGENTS.md` / `CLAUDE.md` / `SYSTEM.md` | Eager, always             | Every turn, loaded at startup                                                                |
| Prompt templates (`/name`)              | Eager, on user invocation | User types `/templatename`                                                                   |
| Skills (Agent Skills standard)          | Lazy, on model decision   | Model reads `SKILL.md` via the `read` tool after seeing the description in the system prompt |

Skills are the interesting one: only skill names and descriptions live in the system prompt; full instructions load on-demand when the model decides the skill is relevant. This is the theory's **progressive disclosure** PromptLoading variant, implemented as a filesystem convention rather than a framework feature. The `bash` tool provides the loading mechanism; the filesystem provides the storage; the model provides the routing. No custom plumbing.

Extensions can also inject into context dynamically via the `context` event or by appending `CustomMessageEntry` records. This is a fourth PromptLoading tier: _programmatic, per-turn_.

### StateMachine — By Extension, Not By Default

PI has no plan mode. The README is explicit: "Write plans to files, or build it with extensions, or install a package." This is the theory's [StateMachine](../patterns/state-machine.md) pattern handled by the extension substrate — if you want phases with different tool capabilities and transition rules, you write an extension that composes Guardrail policies + a phase variable, filters tools based on state, and persists state via `pi.appendCustomEntry()`.

This is the opposite of LangGraph's approach (where StateMachine is the core abstraction). PI treats StateMachine as a composition surface, not as a first-class framework API — consistent with the theory's position that StateMachine is a pattern most coding agents never need. Single-phase tool-use loops cover the default case; when phases are needed, they are domain-specific enough that a fixed DSL would constrain more than it helps.

### Memory — Compaction Is Session-Local, Not Memory

PI's compaction is lossy summarization of older messages within the active tree path when the context window fills up. The full history stays in the JSONL file; `/tree` can revisit it. Crucially, compaction does not write to a separate [Memory](../patterns/memory.md) store — it is a Session-internal transform that replaces a prefix of messages with a `CompactionEntry` containing a summary.

PI has no cross-session Memory pattern implementation beyond `AGENTS.md`. Users are expected to maintain `AGENTS.md` by hand, or build a memory extension. This is conservative — the Memory pattern is an area where the ecosystem is thin, and PI does not pretend to solve it. It provides the substrate (custom entries, extensions, compaction hooks) for someone else to build the Memory layer they want.

### Channel — Single Session, Single Channel (Implicit)

PI is a single-agent system. No [Channel](../patterns/channel.md) variants are exposed because there is no inter-agent communication. Sub-agents are explicitly listed as "not built-in, build with extensions or spawn pi instances via tmux." If you spawn two PI processes and pass data between them, you pick your own channel (filesystem, stdout, git, mailbox) — the framework does not prescribe one.

### Agent — One Core, Many Frontends

The book's [Agent](../harness/agent.md) harness says Frontend + Session + [Environment](../environment/environment.md) are pluggable around an invariant cognitive core. PI demonstrates this concretely — the same Session runs behind multiple Frontends as different Agents:

| Frontend                   | Invocation           | I/O shape                        |
| -------------------------- | -------------------- | -------------------------------- |
| Interactive TUI            | keystrokes via stdin | TUI rendering to stdout (pi-tui) |
| Print mode (`-p`)          | argv + piped stdin   | stdout (final text only)         |
| JSON mode (`--mode json`)  | argv                 | stdout (JSONL event stream)      |
| RPC mode (`--mode rpc`)    | JSONL on stdin       | JSONL on stdout                  |
| SDK (`createAgentSession`) | function call        | `AsyncGenerator<Event>`          |

Same Session. Same AgentLoop. Same extensions and skills. The cognitive core is invariant; only the Frontend changes. This is exactly the decomposition the Agent harness predicts.

---

## The Composition Hierarchy

PI makes the composition hierarchy explicit as a UX surface:

1. **Primitives and harness** (built into `pi-ai` and `pi-agent-core`): LLM + Prompt (`pi-ai`), AgentLoop, Session, Guardrail hooks, event stream.
2. **Agent wrapping** (built into `pi-coding-agent`): Frontend(s) wrapping the Session and pinning it to an Environment — the interactive TUI, print mode, JSON mode, RPC mode, and SDK Frontends all wrap the same Session.
3. **Patterns** (built into `pi-coding-agent`): PromptLoading flavors (AGENTS.md / skills / templates), compaction, session-graph navigation (Workspace-shaped over Sessions).
4. **Compositions by user** (extensions, skills, templates, themes, `pi` packages): plan mode (StateMachine pattern), sub-agents (Topology pattern), MCP bridges (Tool adapters), permission popups (Guardrail policies), git checkpointing, path protection, custom UIs, games during waits.

The distribution mechanism — `pi install npm:@foo/pi-tools` or `pi install git:github.com/user/repo` — promotes compositions to shareable units: capabilities, not applications, are the units of distribution. `pi` packages are to PI what browser extensions are to a browser, but with the crucial difference that a `pi` package can reach _every seam_ — tools, guards, context, commands, UI, provider registration, lifecycle.

---

## What PI Reveals

Insights that generalize beyond PI.

### 1. The Session-Graph Is Navigable State

Individual [Sessions](../harness/session.md) are linear — that does not change. What PI demonstrates is that when users need cheap branching (explore an alternative, keep the abandoned path reachable, return to an earlier point), the right shape is not a bigger Session but a set of sessions that share prefixes, exposed as a graph with a movable pointer. Each leaf is still a linear history; the refinement is that the _relation_ between sessions (shared ancestry) becomes first-class state the user can navigate.

- `/tree` = move the pointer between leaves (switch active session)
- `/fork` = create a new session whose history is a prefix of another
- `/clone` = duplicate the active path
- `branch_summary` = record the gist of an abandoned path before leaving
- `label` = bookmark a node

Most coding agents hide the session-graph — fork is a full copy, sessions are isolated, shared ancestry is invisible. PI makes it the primary UX surface. This is a [Workspace](../patterns/workspace.md) over Sessions, not a change to the Session harness.

### 2. Every Concept Boundary Should Be a User Hook

PI's extension API is organized around lifecycle events at each concept boundary: `tool_call` and `tool_result` at the Guardrail/Tool seam, `context` at the Prompt-assembly seam where PromptLoading plugs in, `before_provider_request` at the LLM seam, `session_start`/`session_shutdown`/`session_compact` at the Session seam, `resources_discover` at the Skill/Template seam, `model_select` at the LLM seam. This is the [seams](../verification/seams.md) claim — every concept boundary is a seam — turned into an extension contract. If you decide your framework's concept decomposition, you get the extension API almost for free: one hook per boundary.

### 3. Progressive Disclosure Beats Preloaded Capability

Skills encode the pattern where only the name and description live in the system prompt, and the full SKILL.md loads when the model decides to use it. This is cheaper (token-wise) and more scalable than stuffing all capabilities into the system prompt. The mechanism is not novel — it is the model reading a file with the `read` tool — but the _convention_ (Agent Skills standard, `/skill:name` invocation, `disable-model-invocation` frontmatter) is what makes it reliable. Conventions over machinery.

### 4. "Primitives, Not Features" Is a Distribution Strategy, Not Just an Architecture

Three constraints make PI's extension substrate actually work:

- **Auto-discovery from conventional paths** (`~/.pi/agent/extensions/`, `.pi/extensions/`). You drop a file and it runs.
- **TypeScript via jiti** (no compile step for authors).
- **npm + git installation with version pinning**. Extensions travel at the speed of a package manager.

Without these, "extensible" degrades into "theoretically extensible, practically nobody writes them." PI invests in the distribution layer as heavily as the API layer. The equivalent concept in browsers is WebExtensions + the extension store; in editors it is VS Code's extension marketplace. PI is building that ecosystem for coding agents.

### 5. Steering and Follow-Up Are an Out-of-Band User Channel

The Session model in the book implicitly assumes turn-based: user speaks, agent speaks, repeat. PI exposes two out-of-band user channels:

- **Steering** (Enter while tools run) — delivered after the current assistant turn's tools finish.
- **Follow-up** (Alt+Enter) — delivered after the agent would have otherwise stopped.

The user is not a turn participant; the user is a real-time controller with a write channel into the Session that fires at well-defined barriers. This is a refinement the book currently lacks: the user-to-agent channel is asymmetric and has its own scheduling semantics.

### 6. Many Shells from One Core Validate the Agent Harness

The same `Agent` object (PI's internal class) powers interactive TUI, print mode, JSON event stream, RPC-over-stdio, and SDK embedding. No branching in the cognitive code. This is the strongest empirical evidence for the [Agent](../harness/agent.md) harness as defined: if the Frontend is a clean boundary, one cognitive core should serve every frontend with zero modification. PI does.

---

## What the Theory Reveals About PI

Insights flowing from the book back to PI users.

### 1. Compaction Is a PromptLoading Transform, Not a Memory Write

PI treats compaction as a Session-local operation: it summarizes the current path and discards detail, producing no artifact visible to future sessions. The theory classifies this precisely: compaction sits at the Prompt-assembly seam (PI's `transformContext` hook), replacing a prefix of Session.history with a summary before the prefix is snapshotted into Prompt.messages. It is a [PromptLoading](../patterns/prompt-loading.md) transform (dynamic strategy over the Session history source), not a [Memory](../patterns/memory.md) write. Users who want their "compacted learnings" to survive across sessions need a separate Memory mechanism — `AGENTS.md` updates, a custom extension that writes to disk at compaction time, or equivalent. The book clarifies the distinction PI's docs imply but don't state.

### 2. PI Has No Memory Layer; AGENTS.md Is Eager PromptLoading

PI's "memory" across sessions is `AGENTS.md`, which the user maintains by hand. This is not [Memory](../patterns/memory.md) as the theory defines it — it is eager [PromptLoading](../patterns/prompt-loading.md) of a file from disk into the system prompt every turn. The theory's Memory taxonomy — cognitive types (episodic, semantic, procedural), write and retrieval strategies, scopes — gives PI users a framework for building the memory layer PI deliberately left out. Anyone building a PI memory extension should use the taxonomy as a checklist: which cognitive type is being stored? Who triggers the write? Who triggers the read?

### 3. Plan Mode, Sub-Agents, and MCP Bridges Are Each a Pattern Composition

PI rejects all three as core features. The theory names what each is:

- **Plan mode** = [StateMachine](../patterns/state-machine.md) with a plan phase and an implement phase, each exposing a different tool set. Extension writes a custom state, filters tools by state, transitions on user command or model signal.
- **Sub-agents** = a small [Topology](../patterns/topology.md) — spawning a child [Session](../harness/session.md) with a derived system prompt and a return [Channel](../patterns/channel.md) (stdout, file, or shared state) back to the parent.
- **MCP bridges** = a [Tool](../primitives/tool.md) adapter that translates MCP server descriptors into PI tool registrations.

None are mysterious. Each is one pattern composition over PI's harness. PI's philosophy says "you can build this"; the theory says _what you would be building_.

### 4. Continuity Is a Cross-Session Pattern PI Doesn't Ship

PI ships Sessions (linear, with a shared-prefix session-graph and compaction) but not [Continuity](../patterns/continuity.md). The Continuity pattern composes Memory + PromptLoading (startup sequence) + Guardrail (clean state) + Tool (bootstrap). PI ships the primitives and harness that Continuity needs, but no Continuity implementation itself. A project spanning weeks on PI needs the user to build their own Continuity layer: an `AGENTS.md` that points to a ground-truth JSON, a bootstrap script, a commit-clean guardrail policy as an extension, and a startup sequence encoded in `AGENTS.md`. PI gives the substrate; the pattern is unwritten.

This is not a gap in PI's theory; it is a conscious trade-off. PI's philosophy asks users to build their workflow. Continuity is one workflow.

### 5. The Session-Graph Is a Partial Solution to the Context Window Boundary

The book identifies Session's hard limitation: "history is bounded by the LLM's context window. When it overflows, the session dies or loses the beginning." PI's session-graph + compaction + `/tree` navigation softens this in three ways:

- **Compaction** trades detail for fit (lossy).
- **`/fork`** lets you branch into a fresh session at any prior point (picks up a sub-path, drops the rest).
- **`branch_summary`** preserves the gist of an abandoned path when switching branches.

But the fundamental limitation remains. The LLM view of any single session is still capped at the context window, regardless of how many sessions coexist in the graph. PI exposes structure around the boundary; it does not remove the boundary. The theory is right that Memory (cross-session) is the real solution, not larger Sessions.

### 6. The "No Permission Popup" Stance Is a Guardrail-vs-Sandbox Composition Argument

PI's explicit argument: run in a container; permission popups belong outside the agent. The theory frames this cleanly: [Guardrails](../harness/guardrail.md) enforce at the tool boundary, sandboxes enforce at the environment boundary, and both layers compose. A container is a sandbox (environment-level). A permission popup is a Guardrail (tool-level). PI's position is that for coding tasks, sandbox is sufficient and Guardrails are the user's choice, so the framework should not impose one. This is defensible — and it is _exactly_ the compositional framing the book prefers.

---

## Why This Framework, From First Principles

The rationale for PI's design choices follows from a small number of premises about what a coding agent is for.

### Premise 1: The Model Is Commodity; Composition Is the Differentiator

If two coding agents using the same frontier model differ by 3× on real tasks, the difference is in tool design, context strategy, and guardrail configuration — not in model selection. A framework that bakes in a particular composition ships a particular opinion. A framework that exposes every composition surface lets the user own the opinion.

PI is the second kind. This only makes sense if you believe the composition — not the model — is where the product lives. The book argues this explicitly in README.md's "Composition Is the Differentiator" section. PI demonstrates it by building the framework that follows from the argument.

### Premise 2: Workflows Are Personal; Agents Must Adapt

A solo founder building Solana trading infra has different needs than a team building a CRUD web app. They need different tools, different guardrails, different context, different permissions. A shipping agent that picks _one_ workflow serves one audience well and every other audience badly. The alternative is to ship the substrate and let each audience compose their own. This is exactly what a package-manager-like distribution layer enables.

The target user for PI is someone who would otherwise fork a coding agent to adjust it. PI removes the fork and replaces it with an extension. The mental model shift is meaningful: you are not customizing someone else's product; you are composing your own from the same building blocks the core ships.

### Premise 3: Agent Behavior Is Configuration, and Configuration Should Hot-Reload

Extensions, skills, prompt templates, and themes are hot-reloadable via `/reload`. A long-running session can evolve its behavior mid-task without restarting. This is the operational consequence of treating config as code: if the agent's behavior is a composition, and the composition is a directory of TypeScript files, hot-reloading the directory hot-reloads the agent.

This matters for the [Continuity](../patterns/continuity.md) pattern: over a multi-week project, the user's understanding of what the agent should do evolves. Hot-reload means the user can improve the agent while using it, without losing session state.

### Premise 4: The Right Concept Decomposition Makes Features Accidental

If your framework has hooks at each concept boundary — primitive, harness, environment — then "plan mode," "sub-agents," "MCP," "permission popups," "todos," "background bash," "git checkpointing," "sandbox execution" — every feature on every other agent's roadmap — is a weekend project. You do not ship these; you ship the _surface on which these are cheap_. This is the expression of the claim that a complete concept set closes under composition: once primitives, harness, and environment are right, features emerge as patterns over them, not from new code in the core.

PI is a bet that this theorem holds in practice. The empirical evidence so far (a working Doom extension, a working sub-agent extension, a working permission gate, a working Claude-Code lookalike theme) suggests the bet is paying off.

### Why It's Worth Exploring

For a developer who:

- Builds custom tooling habitually (you already write extensions to everything you use),
- Thinks in systems (you want to understand the decomposition, not memorize the API),
- Runs in environments where one-size-fits-all permission models are wrong (containers, remote SSH, production debugging, ephemeral VMs),
- Values compositional clarity over feature count,

PI is the coding agent whose architecture reflects how you already think. It does not save you from making the composition choices; it forces you to make them. For someone comfortable making them, that is a feature, not a cost. For someone who wants a turnkey product, it is explicitly the wrong choice and the README says so: "pi ships with powerful defaults but skips features like sub agents and plan mode. Instead, you can ask pi to build what you want or install a third party pi package that matches your workflow."

---

## Summary

PI is what you get when you design a coding agent from the same concepts the theory enumerates, without cheating by bundling. Its packages align with the theory's tiers: `pi-ai` implements [LLM](../primitives/llm.md) + [Prompt](../primitives/prompt.md) (unified provider-agnostic format); `pi-agent-core` is the [AgentLoop](../harness/agent-loop.md) + [Guardrail](../harness/guardrail.md) harness with hooks at every concept boundary; `pi-coding-agent` is the [Agent](../harness/agent.md) harness wrapping a [Session](../harness/session.md) behind multiple Frontends. Extensions, skills, prompt templates, and `pi` packages are the distribution layer that turns the compositional surface into an ecosystem.

The theory decomposes what PI exposes; PI exposes what the theory decomposes. They agree about the concepts and disagree only about emphasis: the theory is a map, PI is a worked example. The refinements PI suggests — the session-graph as a navigable Workspace over linear Sessions, the out-of-band user channel for steering/follow-up, compaction as a PromptLoading transform rather than a Memory write, many Frontends over one cognitive core — belong in the book. The gaps PI leaves — Memory, Continuity, StateMachine for complex workflows, Topology for multi-agent — are where the book continues beyond any single-agent coding tool.

---

## Related

- [LangGraph case study](langgraph.md) — the bundled-framework contrast: LangGraph decides the composition; PI refuses to.
- [AgentLoop](../harness/agent-loop.md) — the while loop at PI's core
- [Session](../harness/session.md) — Sessions stay linear; PI exposes a navigable graph of sessions sharing prefixes
- [Workspace](../patterns/workspace.md) — the session-graph is a Workspace over Sessions
- [PromptLoading](../patterns/prompt-loading.md) — AGENTS.md, skills, prompt templates across a laziness spectrum
- [Guardrail](../harness/guardrail.md) — PI's `beforeToolCall`/`afterToolCall` hooks
- [Continuity](../patterns/continuity.md) — what PI deliberately does not ship, and what a PI user building a multi-week project must build on top
- [Agent](../harness/agent.md) — one cognitive core, many Agents (interactive TUI, print, JSON, RPC, SDK) built from the same Session
- [Verification](../verification/index.md) — every concept boundary is a seam; PI turns every seam into an extension point
