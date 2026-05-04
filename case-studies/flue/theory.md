# Flue Against the Theory: PI + Deploy Layer

Flue is built directly on top of PI's two core packages: [`@mariozechner/pi-ai`](../pi.md) (LLM/Prompt unification) and `@mariozechner/pi-agent-core` (the AgentLoop with Guardrail hooks). The [PI case study](../pi.md) decomposes that lower half against the theory; the same decomposition applies unchanged to Flue. What follows is what Flue _adds_ on top.

## Inherited from PI

[LLM](../../primitives/llm.md) and [Prompt](../../primitives/prompt.md) primitives via `pi-ai` (15+ providers, one stream API). The [AgentLoop](../../harness/agent-loop.md) while-loop and [Guardrail](../../harness/guardrail.md) hooks (`beforeToolCall`/`afterToolCall`) via `pi-agent-core`. Linear [Session](../../harness/session.md) history. Compaction as a Session-local [PromptLoading](../../patterns/prompt-loading.md) transform.

## Added by Flue

1. **[Frontend](../../harness/agent.md) = HTTP service.** PI ships five Frontends (TUI / print / JSON / RPC / SDK) wrapping one running cognitive core in a single process. Flue's Frontend is structurally different: a generated HTTP server with route-per-agent webhook triggers (and cron manifest entries) declared in the agent file and emitted by the build. The deploy unit _is_ the harness.

2. **[Environment](../../environment/environment.md) as a runtime-pluggable substrate (`SessionEnv`).** PI assumes the local Environment (host filesystem + processes). Flue introduces a uniform `exec + fs + cwd + scope` interface with four adapters — in-memory just-bash, host mount, `BashFactory`, and external `SandboxFactory` connectors (Daytona / Vercel / CF Containers). In theory terms, Environment becomes a _parameter_ of the Agent rather than a fixed property of the host process, which is what makes "write once, deploy on Node / Cloudflare / GitHub Actions" tractable.

3. **[Topology](../../patterns/topology.md) baked in (`task` subagents).** PI explicitly leaves Topology out — _"no sub-agents; build with extensions or spawn pi instances via tmux."_ Flue ships it: `session.task()` spawns a child Session with derived context, depth-bounded to 4, and the LLM can autonomously delegate via the built-in `task` tool. One of PI's deliberate gaps, closed.

4. **A typed-eager Skill invocation contract.** PI Skills are lazy progressive-disclosure (model decides to read `SKILL.md`). Flue keeps lazy Skills _and_ adds an eager call-site form: `session.skill(name, { args, result: schema })` returns valibot-validated structured output. A fifth [PromptLoading](../../patterns/prompt-loading.md) flavor — eager-on-user-invocation with a typed contract — that PI doesn't expose.

5. **Roles as a built-in multi-persona substrate.** Build-time-bundled markdown files in `roles/` with frontmatter (description, optional model pin). Roles compose into `<role>...</role>` system-prompt blocks with `per-call > session > agent` precedence. PI has no native role concept — you'd build it with extensions.

6. **Pluggable persistence + multi-tenancy.** PI persists sessions as local JSONL (the shared-prefix session-graph). Flue parameterizes persistence via `SessionStore`: `InMemorySessionStore` on Node, **Durable Object SQLite** on Cloudflare so a session resumes on the same DO across requests, weeks later. Session ID arrives in the request path — multi-tenancy is structural, not a usage convention. A real substrate for the [Continuity](../../patterns/continuity.md) pattern, even though Flue doesn't ship Continuity itself.

7. **Per-call scoped runtime overrides.** Tools, model, system prompt, role, and commands are swappable for one `prompt` / `skill` / `task` call via a snapshot/restore wrapper around `pi-agent-core`'s `Agent`. PI's Agent state is more monolithic; per-call tool sets require extensions. Flue makes the swap a structural feature so the same Session can serve different roles/models on different turns without forking.

8. **Provider precedence chain.** Inherits `pi-ai` but adds `provider/model-id` strings + per-provider runtime settings (`baseUrl` / `headers` / `apiKey`) with `per-call > role > agent` precedence. Makes API gateways (LiteLLM, enterprise proxies) first-class.

9. **AI-generated connectors as a distribution model.** PI distributes capabilities as `pi install npm:@foo/pi-tools` — npm/git package boundaries. Flue's `flue add <provider-url>` pipes a markdown spec to a coding agent (Claude/Codex), which writes the `SandboxFactory` adapter into the user's project. The extension contract isn't a runtime API — it's a _prompt-to-an-agent_ API. Distribution shifts from "ship a package" to "ship a spec the agent compiles into your repo."

## Same gaps as PI

No [StateMachine](../../patterns/state-machine.md) / plan mode (single-phase loop). No first-class [Memory](../../patterns/memory.md) — `AGENTS.md` is eager [PromptLoading](../../patterns/prompt-loading.md) from the sandbox cwd, not Memory in the theory's sense. No [Continuity](../../patterns/continuity.md) implementation, though `SessionStore` + DO SQLite + sandbox-mounted markdown finally provides the substrate to build it on.

## One-line position

If PI is _"primitives, not features"_ applied to a single-user coding agent, **Flue is the same primitives repackaged as a multi-tenant deploy unit, closing the gaps PI left out for product reasons (Topology, persistent multi-tenancy, runtime-agnostic Environment) and accepting the gaps PI left out on principle (StateMachine, Memory).**
