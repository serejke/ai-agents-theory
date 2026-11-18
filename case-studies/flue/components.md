<a id="high-level-components-of-flue"></a>

# The components of Flue

Flue has eight components. This document puts them in groups by function, not by
file. Some components are in two or more modules. One component, the connector
system, is not in `packages/`.

<a id="1-workspace-model--build-pipeline"></a>

## 1. The project folder and the build pipeline

_`build.ts`, `build-plugin-node.ts`, `build-plugin-cloudflare.ts`,
`cloudflare-wrangler-merge.ts`_

This component is the contract of the build. It tells which files a Flue project
has on disk: `agents/`, `roles/`, an optional `.agents/skills/` and `AGENTS.md`.
It compiles these files into an output that you can deploy.

The build does not put `AGENTS.md` and the skills into the output. This is the
most important decision of the build. Flue finds these files at runtime, in the
folder of the Session. Thus the knowledge of an agent can be on R2, and an agent
can operate on a live filesystem.

The build finds the agents in the files that export a `triggers` object. It
makes the output with a `BuildPlugin` interface, which has two modes:

- `'esbuild'` is for Node. It writes a `server.mjs` file.
- `'none'` is for Cloudflare. It writes the source of the entry, and `wrangler`
  makes the bundle. This mode prevents a second bundle of the code with
  `nodejs_compat`.

<a id="2-runtime-entry--trigger-layer"></a>

## 2. The runtime entry and the trigger layer

_the `_entry.ts` that the build writes, `internal.ts`, `errors.ts`,
`error-utils.ts`_

The build writes a server shell for each target. This shell does these tasks:

1. It connects each webhook trigger to an HTTP endpoint.
2. It reads and examines the request to the agent.
3. It makes the `FlueContext` with `createFlueContext`.
4. It sends the request to the handler of the user.
5. It sends the steps of the agent to the client with SSE.

The entry imports its items from `internal.ts`, which is a small and stable
module. Flue keeps this module small by decision. Thus `pi-ai` is not a
dependency in the package manifest of a user project.

Cron triggers are only entries in the manifest here. The platform starts them,
for example the cron of Cloudflare or the cron of GitHub Actions.

<a id="3-session-harness"></a>

## 3. The Session harness

_`session.ts`, `agent-client.ts`, `client.ts`_

This component is the core of Flue. `client.ts::createFlueContext` makes the
`FlueContext` that the handlers get. `init()` finds the sandbox, finds the
context and makes an `AgentClient`, which is the `FlueAgent`. The `AgentClient`
keeps a map of Sessions. Each `Session` contains one `Agent` of `pi-agent-core`
and adds these functions.

### Calls with changes for one call

A Session has the calls `prompt`, `skill` and `task`. Each call goes through
`withScopedRuntime`. During the call, this function changes the `model`, the
`systemPrompt` and the `tools` of the inner harness. At the end of the call, it
puts back the previous values. Thus a call can change the role, the model, the
tools and the commands, and Flue does not make a second copy of the `Agent`.

### Task subagents

`task()` starts a new Session for a subagent. This Session has a different
context: a different folder, a different role and a different history. The
maximum depth of these Sessions is 4. The LLM can use the same mechanism with
the `task` tool.

### Results with a schema

A call with `result: v.object(...)` starts a second prompt that gets the result
(`result.ts`). If the result does not agree with the schema, Flue tries again.

<a id="4-sandbox-substrate-sessionenv"></a>

## 4. The sandbox layer: `SessionEnv`

_`types.ts`, `sandbox.ts`, `env-utils.ts`, `cloudflare/cf-sandbox.ts`,
`cloudflare/virtual-sandbox.ts`_

`SessionEnv` is one interface that each type of sandbox supplies. It holds
`exec`, `readFile`, `writeFile`, `stat`, `readdir`, `mkdir`, `rm`, `cwd`,
`resolvePath`, `scope` and `cleanup`. This interface is the most important part
that lets Flue run on each runtime. The other parts of the SDK do not know which
sandbox they use.

Flue has four adapters for this interface:

- **Empty (`virtual`).** A `just-bash` shell in memory, with no access to the
  host. This is the default. It uses a small number of resources and it is fast.
  Thus it is good for agents that get many requests.
- **Local.** The filesystem of the host at `/workspace`. Only Node has it.
- **`BashFactory`.** A function of the user gives new `Bash` shells. The shells
  use the same filesystem through a closure.
- **`SandboxFactory`.** External sandboxes through one adapter point, for
  example Daytona, Cloudflare Containers and Vercel.

`createCwdSessionEnv` gives a `SessionEnv` that can use only one folder.
`createScopedEnv` gives a `SessionEnv` with the commands of one call.

<a id="5-tool-surface"></a>

## 5. The tools

_`agent.ts`, `mcp.ts`, `command-helpers.ts`, `node/define-command.ts`,
`cloudflare/define-command.ts`_

The tools are the operations that the LLM can do. They come from four sources.
At the boundary of `pi-agent-core`, all the tools become one `AgentTool[]` list.

- **The tools of Flue** (`createTools`): `read`, `write`, `edit`, `bash`,
  `grep`, `glob` and `task`. They are the same as the tools of Claude Code by
  decision. Thus a prompt operates in the two systems.
- **Your `ToolDef`s.** Each `ToolDef` has a type in `TypeBox` or JSON Schema.
  You add it for all the agent with `init({ tools })` or for one call. Flue
  compares its name with `BUILTIN_TOOL_NAMES`. Thus two tools cannot have the
  same name.
- **`Command`s.** Small CLI adapters that Flue adds to `just-bash` during one
  call. They let the LLM run `gh`, `npm` and other CLIs, and the secrets of the
  host stay on the host. A `defineCommand` function for each platform changes a
  CLI of the host into a `Command`.
- **MCP.** `connectMcpServer` finds the tools of an MCP server and gives
  `ToolDef`s that the SDK can add.

<a id="6-context--behavior-layer"></a>

## 6. The context layer

_`context.ts`, `roles.ts`, `internal.ts::resolveModel`_

This component keeps the instructions of the agent in Markdown. When the Session
starts, Flue finds the system prompt and the skills in the sandbox. It finds
them in the folder of the Session, not in the output of the build. The build
puts the roles into the output.

- **System prompt.** `AGENTS.md`, and then `CLAUDE.md` if the project has it.
- **Skills.** Each skill is a `.agents/skills/<name>/SKILL.md` file. Its
  frontmatter gives the `name` and the `description`. Its text after the
  frontmatter becomes a prompt with a structure. You call it with
  `session.skill(name, { args, result })`. Flue can also find a skill by its
  path. Thus a skill can have a top `SKILL.md` file and one file for each step
  in the same folder.
- **Roles.** Each role is a `roles/*.md` file that the build puts into the
  output. Its frontmatter can select a `model`. Each role becomes a
  `<role name="...">…</role>` block of the system prompt.

Flue selects the model, the role and the provider in this sequence:

1. the value of the call
2. the value of the role
3. the default value of the agent.

`ProvidersConfig` (`baseUrl`, `headers`, `apiKey`) uses the same sequence.
`pi-ai::getModel` finds the model from a `provider/model-id` string. The
`applyProviderSettings` function lets you send all model calls through a server
between the agent and the providers.

<a id="7-persistence--compaction"></a>

## 7. Storage and compaction

_`session-history.ts`, `compaction.ts`, `cloudflare/session-store.ts`,
`InMemorySessionStore`_

This component keeps the conversations. `SessionData` has a version
(`version: 2`). It holds `SessionEntry` entries with a type: `message`,
`compaction` or `branch_summary`. Each entry has a `parentId` that connects it
to the entry before it. These connections make an active path through the
history. Thus the format has branches as a basic part.

Flue has two backends for storage:

- on Node, the memory of the process
- on Cloudflare, the SQLite database of a Durable Object. A Session can continue
  on the same Durable Object after some weeks.

`compaction.ts` starts a compaction in two modes:

- **At the limit.** The tokens are more than `contextWindow - reserveTokens`.
- **After an error.** The provider gives an error because the context is too
  large. Flue does a compaction and tries the call again automatically.

A compaction changes the messages before them into one short `compaction` entry.
It keeps the last `N` tokens.

<a id="8-developer-toolchain"></a>

## 8. The developer toolchain

_`packages/cli/bin/flue.ts`, `dev.ts`, `connectors/*.md`,
`scripts/generate-connector-index.ts`_

This component is the thin outer shell. The CLI has the commands `dev`, `run`,
`build` and `add`.

`dev.ts` has two different methods to load changed code again, because the two
targets are very different:

- Node has no tool that makes a bundle. Thus `dev` runs `esbuild`. After each
  change, `dev` stops the server process and starts it again.
- On Cloudflare, `dev` lets `wrangler` find the changes. `dev` changes the entry
  only when the structure changes: when you add or remove an agent, or when you
  change `wrangler.jsonc`. If you change only the code of an agent, `wrangler`
  loads the new code.

The connector system (`flue add`) is the most unusual component. Flue sends a
Markdown specification to a coding agent, for example Claude or Codex. The
coding agent writes the code of the connector into the project of the user. Thus
a connector does not come from a plugin API of the runtime. It comes from a
prompt to an agent.

---

<a id="how-they-fit-together"></a>

## How the components connect

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

Read this structure from the top:

- Components 1 and 8 operate when you write and compile the agent.
- Component 2 is the boundary that you deploy.
- Component 3 is the core of the runtime.
- Components 4 to 7 are the dimensions of the harness. Each dimension changes
  and the other dimensions stay the same.

The dimensions are these:

- the sandbox: where the work occurs
- the tools: the operations that the LLM can do
- the context: how the agent operates
- the storage: the data that the agent keeps.

Each function of Flue is in one of these dimensions. Thus the framework stays
small, but it has many functions.
