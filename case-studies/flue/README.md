# Case Study: Flue

[Flue](https://github.com/withastro/flue) is a TypeScript agent framework whose compilation unit is an _agent harness_ — the whole runtime environment surrounding an LLM (sandbox, filesystem, tools, context, conversation state) — packaged as a deployable HTTP server. It's authored by the Astro team and originated as the workflow tooling inside the Astro GitHub repo.

The framework targets a specific gap. Most agent SDKs are libraries: you import them, wire up an LLM client, manage session state yourself, choose how to deploy. Flue is structured the other way around — a workspace directory (`agents/`, `roles/`, `.agents/skills/`, `AGENTS.md`) compiles via `flue build` into a self-contained Node bundle or Cloudflare Worker. Persistence, SSE streaming, multi-tenancy via session IDs, webhook/cron triggers, and sandbox provisioning are baked into the artifact, not assembled by the user. The relationship to library-style agent SDKs is the same as Next.js's relationship to Express.

![A triage agent: agents/triage.ts, invocable as `flue run triage` or POST /agents/triage/:id — initializes an agent, calls a skill with a valibot result schema, prompts for a comment, and uses session.shell to run git and gh with scoped env.](images/example.png)

The image above is the canonical shape of a Flue agent. One file, default-export an async handler taking `{ init, payload, env }`, get a session, drive it with `prompt` / `skill` / `shell`. The same file is reachable two ways without changes — `flue run triage` for CLI/CI invocation and `POST /agents/triage/:id` for the deployed HTTP route — because triggers are declared in the file and the build emits the routing.

Three properties make Flue worth examining as a case study:

1. **The harness is the deploy unit, not the agent code.** A Flue project compiles to a server. Your handler is ~10 lines; the surrounding harness — session lifecycle, history, compaction, tool dispatch, sandbox lifecycle, transport — is generated.

2. **Logic lives in markdown.** `AGENTS.md` becomes the system prompt; `.agents/skills/*/SKILL.md` files become invokable workflows; `roles/*.md` become subagent system-prompt blocks with optional model pins. These are read at session-init from inside the sandbox cwd, so they can come from a mounted filesystem (e.g. an R2 bucket on Cloudflare) and change without redeploying.

3. **`SessionEnv` as the universal substrate.** A single `exec + fs + cwd + scope` interface that every sandbox mode implements (in-memory just-bash, host filesystem, BashFactory, external connectors like Daytona / Vercel / CF Containers). Built-in tools, skill loading, custom commands, and shell calls all bottom out here, which is what makes "write once, deploy on Node / Cloudflare / GitHub Actions / GitLab CI" tractable rather than aspirational.

The tool surface itself (`read / write / edit / bash / grep / glob / task`) is deliberately the same as Claude Code's, so prompts and skills written for Claude Code port directly. Internally, Flue wraps `@mariozechner/pi-agent-core`'s `Agent` and uses a _scoped-runtime_ pattern — temporarily swapping the agent's tools/model/system-prompt for the duration of one `prompt` / `skill` / `task` call, then restoring — to implement per-call overrides without forking the underlying loop.

## Against the Theory: Flue = pi-mono + Deploy Layer

Flue is built directly on top of pi-mono's two core packages: [`@mariozechner/pi-ai`](../pi-mono.md) (LLM/Prompt unification) and `@mariozechner/pi-agent-core` (the AgentLoop with Guardrail hooks). The [pi-mono case study](../pi-mono.md) decomposes that lower half against the theory; the same decomposition applies unchanged to Flue. What follows is what Flue _adds_ on top.

**Inherited from pi-mono.** [LLM](../../primitives/llm.md) and [Prompt](../../primitives/prompt.md) primitives via pi-ai (15+ providers, one stream API). The [AgentLoop](../../harness/agent-loop.md) while-loop and [Guardrail](../../harness/guardrail.md) hooks (`beforeToolCall`/`afterToolCall`) via pi-agent-core. Linear [Session](../../harness/session.md) history. Compaction as a Session-local [PromptLoading](../../patterns/prompt-loading.md) transform.

**Added by Flue:**

1. **[Frontend](../../harness/agent.md) = HTTP service.** pi-mono ships five Frontends (TUI / print / JSON / RPC / SDK) wrapping one running cognitive core in a single process. Flue's Frontend is structurally different: a generated HTTP server with route-per-agent webhook triggers (and cron manifest entries) declared in the agent file and emitted by the build. The deploy unit _is_ the harness.

2. **[Environment](../../environment/environment.md) as a runtime-pluggable substrate (`SessionEnv`).** pi-mono assumes the local Environment (host filesystem + processes). Flue introduces a uniform `exec + fs + cwd + scope` interface with four adapters — in-memory just-bash, host mount, `BashFactory`, and external `SandboxFactory` connectors (Daytona / Vercel / CF Containers). In theory terms, Environment becomes a _parameter_ of the Agent rather than a fixed property of the host process, which is what makes "write once, deploy on Node / Cloudflare / GitHub Actions" tractable.

3. **[Topology](../../patterns/topology.md) baked in (`task` subagents).** pi-mono explicitly leaves Topology out — _"no sub-agents; build with extensions or spawn pi instances via tmux."_ Flue ships it: `session.task()` spawns a child Session with derived context, depth-bounded to 4, and the LLM can autonomously delegate via the built-in `task` tool. One of pi's deliberate gaps, closed.

4. **A typed-eager Skill invocation contract.** pi-mono Skills are lazy progressive-disclosure (model decides to read `SKILL.md`). Flue keeps lazy Skills _and_ adds an eager call-site form: `session.skill(name, { args, result: schema })` returns valibot-validated structured output. A fifth [PromptLoading](../../patterns/prompt-loading.md) flavor — eager-on-user-invocation with a typed contract — that pi doesn't expose.

5. **Roles as a built-in multi-persona substrate.** Build-time-bundled markdown files in `roles/` with frontmatter (description, optional model pin). Roles compose into `<role>...</role>` system-prompt blocks with `per-call > session > agent` precedence. pi-mono has no native role concept — you'd build it with extensions.

6. **Pluggable persistence + multi-tenancy.** pi-mono persists sessions as local JSONL (the shared-prefix session-graph). Flue parameterizes persistence via `SessionStore`: `InMemorySessionStore` on Node, **Durable Object SQLite** on Cloudflare so a session resumes on the same DO across requests, weeks later. Session ID arrives in the request path — multi-tenancy is structural, not a usage convention. A real substrate for the [Continuity](../../patterns/continuity.md) pattern, even though Flue doesn't ship Continuity itself.

7. **Per-call scoped runtime overrides.** Tools, model, system prompt, role, and commands are swappable for one `prompt` / `skill` / `task` call via a snapshot/restore wrapper around pi-agent-core's `Agent`. pi-mono's Agent state is more monolithic; per-call tool sets require extensions. Flue makes the swap a structural feature so the same Session can serve different roles/models on different turns without forking.

8. **Provider precedence chain.** Inherits pi-ai but adds `provider/model-id` strings + per-provider runtime settings (`baseUrl` / `headers` / `apiKey`) with `per-call > role > agent` precedence. Makes API gateways (LiteLLM, enterprise proxies) first-class.

9. **AI-generated connectors as a distribution model.** pi-mono distributes capabilities as `pi install npm:@foo/pi-tools` — npm/git package boundaries. Flue's `flue add <provider-url>` pipes a markdown spec to a coding agent (Claude/Codex), which writes the `SandboxFactory` adapter into the user's project. The extension contract isn't a runtime API — it's a _prompt-to-an-agent_ API. Distribution shifts from "ship a package" to "ship a spec the agent compiles into your repo."

**Same gaps as pi-mono.** No [StateMachine](../../patterns/state-machine.md) / plan mode (single-phase loop). No first-class [Memory](../../patterns/memory.md) — `AGENTS.md` is eager [PromptLoading](../../patterns/prompt-loading.md) from the sandbox cwd, not Memory in the theory's sense. No [Continuity](../../patterns/continuity.md) implementation, though `SessionStore` + DO SQLite + sandbox-mounted markdown finally provides the substrate to build it on.

### One-line position

If pi-mono is _"primitives, not features"_ applied to a single-user coding agent, **Flue is the same primitives repackaged as a multi-tenant deploy unit, closing the gaps pi-mono left out for product reasons (Topology, persistent multi-tenancy, runtime-agnostic Environment) and accepting the gaps pi-mono left out on principle (StateMachine, Memory).**

## Documents in this case study

- **[components.md](components.md)** — the eight high-level components grouped by responsibility (build pipeline, runtime entry, session harness, sandbox substrate, tool surface, context layer, persistence/compaction, developer toolchain). Read this first for the structural map.
- **[pseudocode.md](pseudocode.md)** — the conceptual architecture rendered as pseudocode: build pipeline, generated entry, session harness loop, the scoped-runtime mechanism, sandbox abstraction, task subagents, compaction. Read this when you want to see the _shape_ end-to-end.

## In one sentence

A build system that turns a markdown-defined workspace into a runtime-agnostic, sandboxed, multi-session HTTP service wrapping pi-agent-core, with the same tool surface and ergonomics as Claude Code.
