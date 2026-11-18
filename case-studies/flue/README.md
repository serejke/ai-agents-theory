# Case Study: Flue

[Flue](https://github.com/withastro/flue) is a TypeScript framework that changes
an agent into an HTTP server that you can deploy. The unit that Flue compiles is
the full agent harness. This harness holds all the parts around the LLM: the
sandbox, the filesystem, the tools, the context and the data of the
conversation. The developers of Astro are the authors of Flue. Flue started as
the workflow tools of the Astro repository on GitHub.

## The output is a server, not a library

Most agent SDKs are libraries. You import the library and connect an LLM client.
Then you keep the data of the Session and you select how to deploy.

Flue operates in the opposite direction. You write a project folder with
`agents/`, `roles/`, `.agents/skills/` and `AGENTS.md`. The command `flue build`
compiles this folder into one Node file or one Cloudflare Worker that contains
all its code. The output does these tasks, and the user does not add them:

- It keeps the Sessions in storage.
- It sends each step to the client with SSE.
- It keeps the Sessions of each tenant apart with the Session IDs.
- It starts agents from webhooks and from cron.
- It makes the sandboxes.

Flue has the same relation to agent libraries that the `Next.js` framework has
to the Express library.

## The shape of a Flue agent

![A triage agent in agents/triage.ts. You can start it with flue run triage or
with POST /agents/triage/:id. It starts an agent, calls a skill with a schema
for the result, prompts for a comment, and uses session.shell to run git and gh
with a limited env.](images/example.png)

The image shows the usual shape of a Flue agent:

1. The agent is one file.
2. The file exports a default handler. The handler is an `async` function that
   gets `{ init, payload, env }`.
3. The handler gets a Session.
4. The handler operates the Session with `prompt`, `skill` and `shell`.

You can start the same file with two different methods, with no changes:

- `flue run triage` runs the agent from the CLI or from CI.
- `POST /agents/triage/:id` is the HTTP endpoint of the agent after you deploy
  it.

This is possible because the file declares its triggers, and the build writes
the routing.

<a id="three-properties-make-flue-worth-examining-as-a-case-study"></a>

## The important properties of Flue

### The harness is the unit that you deploy

A Flue project compiles to a server. The unit that you deploy is the harness,
not the agent code. Your handler has approximately 10 lines. Flue writes all the
harness around it:

- the start, the life and the end of the Session
- the history and the compaction
- the tool calls
- the start, the life and the end of the sandbox
- the connection to the client.

### The instructions are in Markdown

Flue gets the instructions of the agent from Markdown files:

- `AGENTS.md` becomes the system prompt.
- Each `.agents/skills/*/SKILL.md` file becomes a workflow that you can call.
- Each file of `roles/` becomes a block of the system prompt for a subagent. The
  role can also select a model for the subagent.

Flue reads `AGENTS.md` and the skills when the Session starts, from the folder
of the Session in the sandbox. Thus these files can come from an external
filesystem, for example an R2 bucket on Cloudflare. You can change them and not
deploy again.

### One interface for all sandboxes

`SessionEnv` is one interface with `exec`, `fs`, `cwd` and `scope`. Each type of
sandbox supplies this interface:

- a `just-bash` shell in memory
- the filesystem of the host
- a `BashFactory`
- external connectors, for example Daytona, Vercel and Cloudflare Containers.

The tools of Flue, the skills, the commands of the user and the shell calls all
go through `SessionEnv`. Thus you can write an agent one time and deploy it on
Node, Cloudflare, GitHub Actions or GitLab CI. This interface makes it possible.

## The tools and the scoped runtime

Flue has the same tools as Claude Code: `read`, `write`, `edit`, `bash`, `grep`,
`glob` and `task`. This is a decision of Flue. Thus the prompts and the skills
of Claude Code operate in Flue with no changes.

Flue puts the `Agent` of `@mariozechner/pi-agent-core` in its code. It uses a
method that it calls the scoped runtime. For one call of `prompt`, `skill` or
`task`, the scoped runtime replaces the tools, the model and the system prompt
of the `Agent`. After the call, it puts back the previous values. Thus each call
can have different values, and Flue does not make a second copy of the loop.

## The relation to PI

Flue uses the packages `@mariozechner/pi-ai` and `@mariozechner/pi-agent-core`
of [PI](../pi.md) directly. From PI, Flue gets these parts:

- one interface for the LLM and the Prompt
- the AgentLoop
- the Guardrail hooks
- the Session and its compaction.

On top of these parts, Flue adds the parts that deploy the agent:

- an HTTP service as the frontend
- an Environment that changes for each runtime
- Topology with `task`
- skill calls that the caller starts, with a result that has a schema
- roles
- a `SessionStore` that keeps the Sessions of many tenants
- changes for one call with the scoped runtime
- connectors that an AI agent writes.

The document [Flue against the theory](theory.md) compares the parts that Flue
gets from PI with the parts that Flue adds. It uses the words of the theory.

## Documents in this case study

- The document [The components of Flue](components.md) gives the eight
  components of Flue, in groups by function. Read it first for a map of the
  structure. The components are these:
  - the build pipeline
  - the runtime entry
  - the Session harness
  - the sandbox layer
  - the tools
  - the context layer
  - the storage and compaction
  - the developer tools.
- The document [Flue in pseudocode](pseudocode.md) shows the structure of Flue
  as pseudocode. Read it to see the full shape from one end to the other end. It
  shows these parts:
  - the build pipeline
  - the entry that the build writes
  - the loop of the Session harness
  - the scoped runtime
  - the sandbox interface
  - the task subagents
  - the compaction.
- The document [Flue against the theory](theory.md) compares Flue with the
  theory and with the [PI](../pi.md) case study. It tells which parts Flue gets
  from PI and which parts it adds. It also tells which parts that PI does not
  have Flue adds, and which of these parts Flue also does not have.

<a id="in-one-sentence"></a>

## Flue in short

Flue is a build system. It changes a project folder of Markdown files into an
HTTP service around `pi-agent-core`. The service runs on different runtimes, in
a sandbox, with many Sessions. It has the same tools and the same method of
operation as Claude Code.
