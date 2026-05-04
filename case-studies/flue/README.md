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

Flue is built directly on top of [pi-mono](../pi-mono.md)'s `@mariozechner/pi-ai` and `@mariozechner/pi-agent-core` — it inherits the LLM/Prompt unification, the AgentLoop, the Guardrail hooks, and Session/compaction, and adds a deploy layer on top (HTTP service Frontend, runtime-pluggable Environment, Topology via `task`, typed-eager skill invocation, Roles, persistent multi-tenant `SessionStore`, per-call scoped overrides, AI-generated connectors). [theory.md](theory.md) walks through what's inherited vs added against the theory's vocabulary.

## Documents in this case study

- **[components.md](components.md)** — the eight high-level components grouped by responsibility (build pipeline, runtime entry, session harness, sandbox substrate, tool surface, context layer, persistence/compaction, developer toolchain). Read this first for the structural map.
- **[pseudocode.md](pseudocode.md)** — the conceptual architecture rendered as pseudocode: build pipeline, generated entry, session harness loop, the scoped-runtime mechanism, sandbox abstraction, task subagents, compaction. Read this when you want to see the _shape_ end-to-end.
- **[theory.md](theory.md)** — Flue cross-referenced against the theory and the [pi-mono](../pi-mono.md) case study: what Flue inherits, what it adds, and which pi-mono gaps it closes vs. accepts.

## In one sentence

A build system that turns a markdown-defined workspace into a runtime-agnostic, sandboxed, multi-session HTTP service wrapping pi-agent-core, with the same tool surface and ergonomics as Claude Code.
