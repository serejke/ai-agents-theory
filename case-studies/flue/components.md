# High-Level Components of Flue

Flue decomposes into eight components. I'm grouping by responsibility, not by file — several of these are spread across multiple modules, and one (the connector system) lives outside `packages/` entirely.

## 1. Workspace Model & Build Pipeline

_`build.ts`, `build-plugin-node.ts`, `build-plugin-cloudflare.ts`, `cloudflare-wrangler-merge.ts`_

The compilation contract. Defines what a Flue project _is_ on disk (`agents/`, `roles/`, optional `.agents/skills/`, `AGENTS.md`) and turns it into a deployable artifact via a `BuildPlugin` interface with two modes: `'esbuild'` (Node — emit a final `server.mjs`) and `'none'` (Cloudflare — emit raw entry source for wrangler to bundle, deliberately avoiding double-bundling against `nodejs_compat`). Discovers agents by scanning files for an exported `triggers` object. Critically: AGENTS.md and skills are _not_ bundled — they're discovered at runtime from the session's cwd, which is what enables R2-mounted knowledge bases and live filesystem agents.

## 2. Runtime Entry & Trigger Layer

_generated `_entry.ts` + `internal.ts` + `errors.ts` + `error-utils.ts`_

The per-target server shell that the build emits. Maps webhook triggers to HTTP routes, parses/validates the agent request, builds the `FlueContext` via `createFlueContext`, dispatches to the user's handler, and streams events as SSE. `internal.ts` is the stable bare-specifier surface the generated entry imports from — kept narrow on purpose so user projects don't need pi-ai as a direct dep. Cron triggers are manifest-only here; the platform (CF Cron, GitHub Actions schedule) does the firing.

## 3. Session Harness

_`session.ts`, `agent-client.ts`, `client.ts`_

The core. `client.ts::createFlueContext` produces the `FlueContext` handlers see. `init()` resolves the sandbox, discovers context, and constructs an `AgentClient` (the `FlueAgent`). `AgentClient` owns a session map; each `Session` wraps one `pi-agent-core` `Agent` and adds:

- **Three call surfaces** — `prompt`, `skill`, `task` — that all funnel through `withScopedRuntime`, which temporarily mutates the inner harness's `model / systemPrompt / tools` for the duration of the call and restores on exit. This is how per-call role/model/tool/command overrides work without forking the Agent.
- **Task subagents** — `task()` spawns a _child Session_ with its own context (separate cwd, role, history), depth-bounded to 4. The built-in `task` tool is the LLM-facing entry point to the same machinery.
- **Schema-typed results** — `result: v.object(...)` triggers a result-extraction sub-prompt (`result.ts`) with retry on validation failure.

## 4. Sandbox Substrate (`SessionEnv`)

_`types.ts`, `sandbox.ts`, `env-utils.ts`, `cloudflare/cf-sandbox.ts`, `cloudflare/virtual-sandbox.ts`_

The universal interface — `exec + readFile/writeFile/stat/readdir/mkdir/rm + cwd + resolvePath + scope + cleanup` — that every sandbox mode implements. Four concrete adapters:

- **Empty / virtual** — in-memory just-bash, no host access (default; cheap, fast, fits the high-traffic agent profile)
- **Local** — host filesystem mounted at `/workspace`, Node only
- **BashFactory** — user-supplied factory returning fresh `Bash` instances; FS shared via closure
- **SandboxFactory** — external sandboxes (Daytona, CF Containers, Vercel, etc.) wrapped through a single adapter point

Plus `createCwdSessionEnv` for cwd-scoping and `createScopedEnv` for command-injection scoping. This is the abstraction that makes the rest of the SDK sandbox-blind, and it's the load-bearing piece of "runtime-agnostic."

## 5. Tool Surface

_`agent.ts`, `mcp.ts`, `command-helpers.ts` + `node/define-command.ts`, `cloudflare/define-command.ts`_

What the LLM can actually do. Four sources, all unified at the pi-agent-core `AgentTool[]` boundary:

- **Built-ins** (`createTools`) — `read / write / edit / bash / grep / glob / task`. Deliberately mirrors Claude Code's tool surface so prompts port over.
- **Custom `ToolDef`s** — typed (TypeBox/JSON Schema), agent-wide via `init({ tools })` or per-call. Validated against `BUILTIN_TOOL_NAMES` for collisions.
- **`Command`s** — CLI shims registered into just-bash for the duration of a call. Lets the LLM invoke `gh`, `npm`, etc., without leaking host secrets. Platform-specific `defineCommand` factories adapt host CLIs into the Command interface.
- **MCP** — `connectMcpServer` discovers tools from an MCP server and yields `ToolDef`s the SDK already knows how to register.

## 6. Context & Behavior Layer

_`context.ts`, `roles.ts`, `internal.ts::resolveModel`_

The markdown-as-logic system. Three things get discovered/resolved at session-init time, all from inside the sandbox cwd (not the bundle):

- **System prompt** — `AGENTS.md` (and `CLAUDE.md` if present), concatenated.
- **Skills** — `.agents/skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`); body becomes a structured prompt template invokable as `session.skill(name, { args, result })`. Path-based fallback supports orchestration packs (parent SKILL.md + sibling stage files).
- **Roles** — `roles/*.md` (build-time, bundled). Frontmatter can pin a `model`. Roles compose into `<role name="...">…</role>` system-prompt blocks.

Plus the **model/role/provider precedence chain**: `per-call > role > agent default`, with `ProvidersConfig` (baseUrl/headers/apiKey) merged the same way. `pi-ai::getModel` resolves `provider/model-id` strings; the `applyProviderSettings` helper makes gateway routing first-class.

## 7. Persistence & Compaction

_`session-history.ts`, `compaction.ts`, `cloudflare/session-store.ts`, `InMemorySessionStore`_

Stateful conversations. `SessionData` is versioned (`version: 2`) and built from typed `SessionEntry` records — `message | compaction | branch_summary` — with `parentId` linkage forming an active path through history (branching is a primitive, not bolted on). Two backends: in-memory on Node, Durable Object SQLite on Cloudflare (so a session resumes on the same DO weeks later). `compaction.ts` wires in two trigger modes: **threshold** (tokens exceed `contextWindow - reserveTokens`) and **overflow** (provider returned context-overflow → compact + auto-retry). Older messages get summarized into a `compaction` entry; recent N tokens preserved.

## 8. Developer Toolchain

_`packages/cli/bin/flue.ts`, `dev.ts`, `connectors/_.md`, `scripts/generate-connector-index.ts`\*

The thin outer shell. CLI exposes `dev / run / build / add`. The interesting piece is `dev.ts`: it implements **two fundamentally different reload models** because the targets are fundamentally different — Node has no host bundler, so dev does the esbuild itself and respawns the child on every change; Cloudflare delegates to wrangler's own watcher and only re-acts on _structural_ changes (agent set or `wrangler.jsonc` edits), letting wrangler hot-reload bodies. The **connector system** (`flue add`) is the most unusual component: extensibility is implemented by piping a markdown spec to a coding agent (Claude/Codex/etc.) which writes the connector code into the user's project. Connectors aren't a runtime plugin API — they're a _prompt-to-an-agent_ API.

---

## How they fit together

```
Build Pipeline (1)            Dev Toolchain (8)
   │ compiles                    │ orchestrates
   ▼                             ▼
Runtime Entry (2) ── builds ──▶ FlueContext
                                 │ init()
                                 ▼
                          Session Harness (3)
                          ┌──────┼───────────────┐
                          │      │               │
                  Sandbox │   Tool Surface (5)   │ Context (6)
                  (4)     │      │               │   (AGENTS.md,
                  exec+fs │      │               │    skills, roles,
                  base    │      │               │    model routing)
                          │      │               │
                          ▼      ▼               ▼
                    Persistence & Compaction (7)
```

Reading this top-down: **(1)+(8) are compile/dev-time, (2) is the deploy boundary, (3) is the runtime nucleus, and (4)–(7) are the four orthogonal axes the harness composes over** — _where things run_ (sandbox), _what the LLM can do_ (tools), _how it behaves_ (context), and _what it remembers_ (persistence). Every feature in Flue lives on exactly one of those axes, which is why the framework stays small despite covering a lot of surface.
