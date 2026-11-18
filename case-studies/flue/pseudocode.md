<a id="flue--conceptual-architecture-in-pseudocode"></a>

# Flue in pseudocode

This document shows the structure of Flue as pseudocode. It shows which part
calls which part, where Flue keeps the data, and where the unusual decisions
are. The pseudocode does not show the types, the error paths and most options.

<a id="build-time-workspace--deployable"></a>

## At build time: from the project folder to the output

The build reads the agents and the roles of the project folder. Then it writes
the output for Node or for Cloudflare.

```
build(workspaceDir, target):
    plugin = target == "node" ? NodePlugin : CloudflarePlugin

    agents = scan(workspaceDir/"agents/*.ts")
                .map(file => parseExports(file))         # name, triggers
    roles  = scan(workspaceDir/"roles/*.md")
                .map(file => parseFrontmatter(file))     # name, desc, model, body

    # NOTE: AGENTS.md and .agents/skills/ are NOT bundled.
    # Discovered at runtime from session cwd.

    entrySource = plugin.generateEntryPoint({ agents, roles, workspaceDir })

    if plugin.bundle == "esbuild":           # Node
        write(distDir/"server.mjs", esbuild(entrySource))
    else:                                    # Cloudflare → wrangler bundles
        write(distDir/plugin.entryFilename, entrySource)
        write(distDir/"wrangler.jsonc", merge(userWrangler, plugin.additions))
```

<a id="runtime-the-generated-entry-point"></a>

## At runtime: the entry that the build writes

The build writes this entry. For each request, the entry makes a `FlueContext`
and calls the handler of the agent. It sends each step to the client with SSE.

```
# This is the file the build emits. Conceptually:

import userAgents from "./agents/*"     # bundled handlers
ROLES = { ... }                          # baked in at build time

server.on(POST "/agents/:name", async req:
    handler = userAgents[req.name]
    payload = parseJson(req.body)

    ctx = createFlueContext(
        id = req.params.id,
        payload = payload,
        env = platformEnv,                          # process.env or Worker env
        agentConfig = { roles: ROLES, resolveModel, providers, ... },
        createDefaultEnv = () => virtualSandbox(),  # empty just-bash
        createLocalEnv   = () => localSandbox(),    # host fs at /workspace
        defaultStore     = platformStore,           # InMemory | DO-SQLite
        resolveSandbox   = platformSandboxHook,     # CF-specific adapters
    )

    ctx.setEventCallback(event => sse.send(event))
    result = await handler(ctx)
    sse.sendFinal(result)
)
```

<a id="the-handler-the-user-wrote"></a>

## The handler of the user

The user writes only this file. It runs in the server of the entry.

```
# .flue/agents/support.ts  (user code, runs inside the server above)

export triggers = { webhook: true }

export default async (ctx):
    agent   = await ctx.init({ model: "anthropic/claude-sonnet-4-6",
                               sandbox: getVirtualSandbox(ctx.env.KB) })
    session = await agent.session()
    return await session.prompt(`Help with: ${ctx.payload.message}`,
                                { role: "triager" })
```

<a id="init--building-the-flueagent"></a>

## `init()`: how Flue makes the `FlueAgent`

`init()` changes the sandbox option into one `SessionEnv`. Then it reads the
system prompt and the skills from the sandbox, not from the output of the build.

```
ctx.init(opts):
    # Resolve sandbox option to a uniform SessionEnv
    env = match opts.sandbox:
        undefined | "empty" -> createDefaultEnv()
        "local"             -> createLocalEnv()
        BashFactory         -> wrapJustBash(opts.sandbox)
        platformHook(opts.sandbox)? -> ...           # CF Containers, Vercel
        SandboxFactory      -> opts.sandbox.createSessionEnv({ id, cwd })

    if opts.cwd: env = scopeToCwd(env, opts.cwd)

    # Markdown-as-logic discovery — from inside the sandbox, not the bundle
    localContext = discoverSessionContext(env):
        return {
            systemPrompt: read(env, "AGENTS.md") + read(env, "CLAUDE.md"),
            skills:       loadAll(env, ".agents/skills/*/SKILL.md"),
        }

    agentConfig = {
        ...baseAgentConfig,
        systemPrompt: localContext.systemPrompt,
        skills:       localContext.skills,
        model:        resolveModel(opts.model, opts.providers),
        role:         opts.role,
        providers:    merge(baseProviders, opts.providers),
    }

    return new AgentClient(id, agentConfig, env, store, eventCb,
                           opts.commands, opts.tools)
```

<a id="agentclient--session-manager"></a>

## `AgentClient`: the Sessions of an agent

The `AgentClient` keeps one `Session` for each Session ID. It loads the data of
a Session from the store when the Session starts.

```
class AgentClient:
    sessions: Map<id, Session> = {}

    session(id="default", opts):
        if sessions[id]: return sessions[id]
        data = await store.load(storageKey(id))
        s = new Session(id, config, env, store, existingData=data,
                        agentCommands, agentTools, sessionRole=opts.role,
                        createTaskSession=this.makeChildSession)
        sessions[id] = s
        return s

    shell(cmd, opts):                     # raw exec, no LLM, no history
        return env.exec(cmd, opts)

    destroy():
        for s in sessions: s.close()
        await env.cleanup()
```

<a id="session--the-harness-nucleus"></a>

## `Session`: the core of the harness

A `Session` contains the `Agent` of `pi-agent-core` and the history of the
Session. It gives the calls `prompt`, `skill`, `task` and `shell`.

```
class Session:
    history: SessionHistory                # versioned entry log w/ parentId chain
    harness: pi_agent_core.Agent           # the underlying LLM loop

    constructor(...):
        self.history = SessionHistory.fromData(existingData)
        self.harness = pi_agent_core.Agent({
            systemPrompt: config.systemPrompt,
            model:        config.model,
            tools:        builtinTools(env, agentCommands) + customTools(agentTools),
            messages:     self.history.buildContext(),
            getApiKey:    p => config.providers[p]?.apiKey,
        })

        self.harness.subscribe(event => emit(translate(event)))   # SSE stream

    # ─── The three call surfaces ─────────────────────────────────
    prompt(text, opts):  return scopedRun(opts, () =>
                            run(buildPromptText(text, opts.result)))

    skill(name, opts):   skill = config.skills[name] or loadSkillByPath(env, name)
                         return scopedRun(opts, () =>
                            run(buildSkillPrompt(skill, opts.args, opts.result)))

    task(text, opts):    return runTask(text, opts)        # see below

    shell(cmd, opts):    result = env.exec(cmd, opts)
                         history.append(shellMessage(cmd, result))
                         await save()
                         return result
```

<a id="the-clever-bit-scoped-runtime"></a>

### The scoped runtime

The scoped runtime lets one call change the tools, the model and the system
prompt of the `Agent`. Flue does not make a second copy of the `Agent`.

```
scopedRun(opts, fn):
    saved = snapshot(harness.{tools, model, systemPrompt})

    harness.model        = resolveModelForCall(opts.model, opts.role)
                            # precedence: call > role > agent default
    harness.systemPrompt = baseSystemPrompt + roleBlock(opts.role)
    harness.tools        = builtinTools(scope(env, opts.commands),
                                        opts.commands, opts.tools, opts.role) +
                           customTools(agentTools + opts.tools)

    try:    return await fn()
    finally: harness.{tools, model, systemPrompt} = saved


run(promptText):
    beforeLen = harness.messages.length
    await harness.prompt(promptText)
    await harness.waitForIdle()                   # drives the tool-call loop

    # New messages → typed history entries
    for msg in harness.messages[beforeLen:]:
        history.append(messageEntry(msg, source="prompt"))
    await store.save(storageKey, history.toData())

    await maybeCompact()
    return opts.result ? extractResult(opts.result) : { text: lastAssistantText() }
```

<a id="compaction--context-window-management"></a>

### Compaction: how Flue keeps the history in the context window

```
maybeCompact():
    tokens = lastAssistantUsage().total

    if tokens > model.contextWindow - reserveTokens:        # threshold mode
        compact()
    # overflow mode handled in the catch path of run():
    # if isContextOverflow(error): compact(); retry once

compact():
    keep = recentMessagesUpTo(keepRecentTokens)
    older = messages[:-len(keep)]
    summary = await summarizerLLM(older)               # separate LLM call
    history.append(compactionEntry(summary, firstKeptId=keep[0].id))
    harness.messages = [systemPrompt, summary, ...keep]
```

<a id="task-subagents--recursive-sessions"></a>

### Task subagents: Sessions that start Sessions

```
runTask(text, opts):                                   # depth-bounded to 4
    if taskDepth >= 4: throw

    childEnv  = opts.cwd ? scopeToCwd(env, opts.cwd) : env
    childRole = opts.role or self.sessionRole or self.config.role

    child = createChildSession(
        parent = self,
        env    = childEnv,
        role   = childRole,
        depth  = taskDepth + 1,
        # No persistence — child is ephemeral
    )

    try:    return await child.prompt(text, { result: opts.result })
    finally: child.close()
```

The `task` tool of the LLM only calls `runTask`. Thus two sources can start a
subagent: the code of the user with `session.task()`, and the LLM when it gives
a task to a subagent.

<a id="the-substrate-sessionenv"></a>

## The sandbox interface: `SessionEnv`

Each type of sandbox supplies this interface:

```
interface SessionEnv:
    exec(cmd, {cwd, env, timeout}) -> {stdout, stderr, exitCode}
    scope({commands}) -> SessionEnv          # operation-scoped runtime
    readFile / writeFile / stat / readdir / exists / mkdir / rm
    cwd, resolvePath(p), cleanup()
```

These functions supply the interface for each type of sandbox:

```
virtualSandbox():    # default — in-memory just-bash, no host
    bash = new Bash({ fs: in_memory_fs() })
    return adaptBashToSessionEnv(bash)

localSandbox():      # Node only — host fs mounted at /workspace
    return adaptHostFsToSessionEnv(process.cwd())

cfSandboxToSessionEnv(durableObjectContainer): ...
daytonaConnector.createSessionEnv({id, cwd}): ...
```

The commands of a call become shell commands in the sandbox with this method:

```
scope(env, commands):
    if env is just-bash:
        scopedBash = freshBashWith(env.fs)
        for cmd in commands:
            scopedBash.registerCommand(cmd.name, args =>
                cmd.execute(args))           # routes back to host CLI
        return adaptBashToSessionEnv(scopedBash)
    else:
        # Remote sandboxes: wrap exec to intercept registered names
        return interceptingProxy(env, commands)
```

<a id="tool-dispatch--what-the-llm-actually-sees"></a>

## The tools that the LLM sees

Each tool is a thin adapter on the `SessionEnv`.

```
builtinTools(env, ...):
    return [
        readTool(env), writeTool(env), editTool(env),
        bashTool(env), grepTool(env), globTool(env),
        taskTool(roles, runTask=self.runTaskForTool),
    ]

# Each tool is a thin adapter — e.g.:
readTool(env) = {
    name: "read",
    parameters: { path, offset?, limit? },
    execute(params):
        if env.stat(params.path).isDirectory: return env.readdir(params.path)
        return truncate(env.readFile(params.path), MAX_LINES, MAX_BYTES)
}

bashTool(env) = {
    name: "bash",
    execute(params): return env.exec(params.command, { timeout })
}
```

Your tools and the MCP tools have the same `AgentTool` shape. Thus the harness
operates all tools with the same method.

<a id="persistence--the-typed-entry-log"></a>

## Storage: the log of entries

```
SessionData = {
    version: 2,
    entries: SessionEntry[],            # message | compaction | branch_summary
    leafId,                              # active branch tip
    metadata, createdAt, updatedAt,
}

SessionStore = {
    save(id, data), load(id), delete(id)
}

# Two implementations:
InMemorySessionStore        # Node default
DurableObjectSqliteStore    # Cloudflare — survives across requests
```

<a id="dev-loop"></a>

## The `dev` loop

On Node, `dev` makes the output again after each change. On Cloudflare,
`wrangler` finds the changes, and `dev` writes the entry again only when the set
of agents changes.

```
dev(target):
    if target == "node":
        loop:
            build()                      # esbuild → dist/server.mjs
            child = spawn("node dist/server.mjs")
            wait_for_change(workspaceDir, debounce=150ms)
            child.kill()

    if target == "cloudflare":
        wrangler.startWorker(dist/_entry.ts)   # wrangler watches imports itself
        loop:
            change = wait_for_change(workspaceDir)
            if structural(change):       # agents added/removed/triggers changed
                regenerate(dist/_entry.ts)
                if wranglerJsoncChanged: wrangler.restart()
            # else: pure body edit — do nothing, wrangler hot-reloads
```

---

<a id="the-shape-in-one-diagram"></a>

## The full shape

```
                    ┌──────────────────────────────────────────┐
   user code ─────► │  ctx.init({ model, sandbox, ... })       │
                    │            │                              │
                    │            ▼                              │
                    │  AgentClient ───── sessions: Map<id,Session>
                    │            │                              │
                    │            ▼                              │
                    │   Session ─── history: SessionHistory ──► SessionStore
                    │     │     ─── harness: pi_agent_core.Agent
                    │     │          │  tools, model, systemPrompt
                    │     │          │  ◄── scopedRun() mutates per-call
                    │     │          ▼
                    │     │     LLM provider (via pi-ai)
                    │     │          │  tool calls
                    │     │          ▼
                    │     │     [ read | write | edit | bash | grep | glob | task ]
                    │     │          │      │      │      │      │      │      │
                    │     │          ▼      ▼      ▼      ▼      ▼      ▼      ▼
                    │     │              SessionEnv  (exec + fs)
                    │     │                   │
                    │     │     ┌─────────────┼─────────────┬───────────────┐
                    │     │     ▼             ▼             ▼               ▼
                    │     │  virtual       local        BashFactory   SandboxFactory
                    │     │ (just-bash)  (host fs)                   (Daytona, CF, …)
                    │     │
                    │     └──► task() ──► child Session (recursive, depth ≤ 4)
                    └──────────────────────────────────────────┘
```

In short, the core of Flue does these steps:

1. It attaches a `SessionEnv`.
2. It gives the `SessionEnv` to an `Agent` of `pi-agent-core`. All the tools of
   this `Agent` operate on that `SessionEnv`.
3. It keeps the messages as a sequence of entries with types. Each entry
   connects to the entry before it.
4. It gives three calls that operate the `Agent`: `prompt`, `skill` and `task`.
   Each call goes through one `scopedRun`. `scopedRun` puts in the tools, the
   model and the role of the call, and then puts back the previous values.

The other parts are around this loop at the core: the build pipeline, the
trigger layer, the connectors and the `dev` server.
