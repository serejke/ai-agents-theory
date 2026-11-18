# Environment

The Environment is the set of resources where an agent does its work. It holds
the files that the agent reads, the processes that it runs, the network that it
connects to and the credentials that it has. The Environment is not a part of
the agent: the tools are the interface, and the Environment is the item that the
tools operate on.

**Shape**: a group of resources — access to code, a runtime, an operating
system, access to the network, credentials and installed software. The
Environment is not a composition of primitives, harness or patterns. It is a
different tier of the theory.

<a id="why-it-matters"></a>

## Value

The Environment sets the limits of the operations that an agent can do. Give two
agents the same [LLM](../primitives/llm.md), the same
[Tools](../primitives/tool.md), the same [Session](../harness/session.md) and
the same system prompt. If their Environments are different, the two agents
operate very differently:

- A coding agent on your computer can do all the operations that your shell can
  do.
- The same agent in a temporary container can change only the items in the
  container.
- The same agent in a sandbox of a browser can use only the items that the
  sandbox shows.

Thus, two agents with the same configuration and different Environments are two
different agents.

The Environment, not the number of tools, sets the limits of an agent. For a
[generative tool](../primitives/tool.md), for example `bash`, the limits are a
function of the Environment: `capability = f(environment)`. For a specific tool,
the Environment is a configuration that does not change. It sets the limits of
the resources that the tool can use.

The Environment is also the area of most decisions about security and about the
correct operation of an agent. [Guardrails](../harness/guardrail.md) apply
limits at the boundary of the tools. The Environment applies limits at the
boundary of the resources. You cannot know all the operations of a generative
tool before they occur. A new container for each run keeps the damage of these
operations small.

<a id="formal-definition"></a>

## Definition

```typescript
type Environment = {
  code?: CodeAccess; // git clone, local files, read-only mount
  runtime?: Runtime; // node, python, docker, databases
  os?: OSAccess; // shell, gh, ssh, kubectl
  network?: NetworkAccess; // the domains and ports that the agent can reach
  credentials?: Credential[]; // API keys, tokens, SSH keys
  filesystem?: FilesystemAccess; // the paths that the agent can read and write
};
```

All the fields are optional, because the resources of Environments are very
different:

- An agent that only makes a plan can have only `code`, with read-only access to
  the repository.
- A coding agent that builds and runs all the parts of a product has `code`,
  `runtime` and `os`.
- An agent that only does reasoning can have an Environment that is almost
  empty.

The Environment is not a composition in the meaning of the theory. It does not
come from primitives and patterns. It is the bottom layer that the primitives
operate on:

- Tools read and change the Environment.
- [Guardrails](../harness/guardrail.md) put limits on the operations at the tool
  boundary. They do this before an operation gets to the Environment.
- A sandbox puts limits on the Environment.

## Environment and generative tools

The relation between a Tool and its Environment changes with the type of the
[Tool](../primitives/tool.md):

- **Specific tools**, for example `file_read`, `db_query` and `gmail_send`. The
  Environment is a configuration that does not change. The tool schemas set the
  limits of the resources that the agent can use. The agent cannot go out of the
  tool set.
- **Generative tools**, for example `bash`, a tool that runs code and a general
  HTTP tool. The Environment is the limit. The agent can install packages, start
  processes and connect to new services. It can do all the operations that the
  Environment lets it do.

```
Specific tools:    capability = Σ(tool schemas)
Generative tools:  capability = f(environment)
```

Thus, the Environment is most important for agents with generative tools.
Compare a coding agent with `bash` in a new container with the same agent on the
computer of the user. The two agents can do very different operations. If you
cannot put limits on the tool, a sandbox puts limits on the Environment.

## Variants

Some Environments give full access to the host. Other Environments are sandboxes
with many limits, and many Environments are between these two points. For each
Environment, you balance two properties:

- the operations that the agent can do
- the control of the damage that the agent can cause.

| Variant                | Access of the agent                     |
| ---------------------- | --------------------------------------- |
| Local operating system | All the resources of the user           |
| Container              | A new Linux system for each task        |
| Virtual machine        | A full Linux system with a kernel       |
| Sandbox                | Only the resources that the sandbox has |
| Host on the network    | A different system through a protocol   |
| API only               | Only calls to APIs through the network  |

<a id="local-os"></a>

### Local operating system

The agent runs as a process on the computer of the user. It has full access to
the files, the installed tools and the credentials of the user.

```typescript
const localOS: Environment = {
  code: localFS("~/projects/repo"),
  runtime: localNode,
  os: localShell,
  credentials: allUserCredentials, // the agent has all that the user has
};
```

- **Examples**: Claude Code, Cursor, PI and other local coding agents.
- **Good**: The agent can do the most operations. It does not have a setup step,
  and it can use all the tools of the user.
- **Bad**: No boundary isolates the agent. It can do all that the user can do.
  It can cause damage by accident to files that are not related to the task.

### Container

A container is a temporary Environment that isolates the agent. The agent gets a
new Linux system for each task. The container comes from an image that gives the
tools, the packages and the version of the operating system.

```typescript
const container: Environment = {
  code: gitClone("owner/repo"),
  runtime: dockerImage("node:20"),
  os: containerShell({ caps: ["NET_ADMIN"] }),
  credentials: [githubToken], // only the credentials that you give
  network: { allow: ["github.com", "npmjs.org"] },
};
```

- **Examples**: Codex of OpenAI, Copilot Workspace of GitHub and coding agents
  that run automatically after each change of the code.
- **Good**: The container isolates the agent. The agent can install packages,
  run servers and break items, and the host does not change. Each task starts in
  a clean condition. A different run of the same task starts in the same
  condition.
- **Bad**: The setup uses time. The data does not stay after the task, if you do
  not export it. The start is slower than the start of a local agent.

### Virtual machine

A virtual machine isolates a full operating system, not only a container. It
uses more resources than a container, but it isolates the agent more.

```typescript
const vm: Environment = {
  code: gitClone("owner/repo"),
  runtime: fullLinuxVM,
  os: vmShell,
  credentials: [scopedDeployKey],
};
```

- **Examples**: e2b, Modal and agents in the cloud that run for a long time.
- **Good**: The virtual machine isolates the agent more than a container,
  because the kernel is the boundary. The agent can run all the programs that a
  Linux host can run.
- **Bad**: A new virtual machine starts slowly. The cost of each task is higher.
  Usually a container is sufficient, and then a virtual machine gives more than
  necessary.

### Sandbox

A sandbox is an Environment with limits that you select. You give the agent less
access than it can have. Each sandbox selects the access that it stops.

```typescript
const codeSandbox: Environment = {
  code: readOnlyFS("/workspace"),
  runtime: pythonOnly,
  os: null, // no shell
  network: { allow: [] }, // no network
  credentials: [], // no credentials
};

const browserSandbox: Environment = {
  // The agent can use only one browser page. No filesystem, no shell.
  browser: chromiumInstance({ singlePage: true }),
  network: { allow: ["*"] },
};
```

- **Examples**:
  - Code Interpreter of ChatGPT. It runs only Python, it has no network, and its
    data does not stay after the task.
  - Sessions of `agent-browser`.
  - Agents of companies that examine documents. They have read-only access to a
    specified tree of documents.
- **Good**: The sandbox gives the most control of the damage. It is correct for
  untrusted data, for automatic operations of a different company and for tasks
  that must obey external rules.
- **Bad**: The agent can do only the operations that the sandbox lets it do. For
  an agent with generative tools, this limit is frequently the correct boundary.
  For an agent with specific tools, a sandbox can give less control than
  Guardrails at the tool level.

A sandbox is not a different concept. It is one point in the range of
Environments. The word "sandbox" shows that you put limits on the Environment
for safety. There is no clear line between "a container in a sandbox" and "a
container with tight rules".

<a id="remote-host-ssh-kubectl-cloud-apis"></a>

### Host on the network

The Environment of the agent is a different system on the network. The agent
connects to it through a protocol, for example SSH, `kubectl` or the APIs of a
cloud. The agent operates on the system, but it does not run in it.

```typescript
const remote: Environment = {
  os: ssh({ host: "prod-server", key: deployKey }),
  runtime: kubectl({ cluster: "prod" }),
  credentials: [deployKey, kubeConfig],
};
```

- **Examples**: agents that operate servers, automatic operations that keep the
  servers of a company available, and agents that control systems on the
  network.
- **Good**: The agent can operate on the servers that the customers use, but it
  does not run on them. The access rules of these servers also apply to the
  agent, for example SSH keys and the access rules of Kubernetes.
- **Bad**: The network adds latency. The data about the operations of the agent
  is on two or more systems. If a Session stops, it is not easy to continue it.

<a id="api-only--network-only"></a>

### API only

The agent has no filesystem and no shell. It can only send calls to APIs through
the network. This variant is the nearest to "only reasoning on external data".

```typescript
const apiOnly: Environment = {
  network: { allow: ["api.github.com", "api.slack.com"] },
  credentials: [githubToken, slackBotToken],
};
```

- **Examples**: agents in Slack, agents that do the steps of a workflow
  automatically, and agents that connect two or three services.
- **Good**: An attacker has the smallest number of points for an attack. The
  agent keeps no data between calls. It is easy to know the operations of the
  agent.
- **Bad**: The agent can do only the operations that the APIs give. It cannot
  calculate on a local computer, and it cannot use files.

<a id="design-implications"></a>

## How to select an Environment

Select the Environment carefully. The default "my computer, full access" is
frequently not correct when other persons use the agent. Use an Environment with
limits if the agent does one of these operations:

- It reads untrusted data.
- It runs code from a different company or person.
- It operates on the systems that customers use.

For these agents, the Environment is the primary control of the damage.
Guardrails add a second layer of control at the tool boundary.

For an agent with generative tools, the Environment sets the correct limit of
its operations. Guardrails on `bash` find the patterns of commands that you know
at the start. The sandbox sets the limits of the operations that the agent can
do, also if it goes around each Guardrail.

Companies also make the Environment a product for their customers. "Give agents
a reliable, isolated Linux in one second" is the message of the cloud
Environments of OpenAI Codex, e2b, Modal and Cloudflare. The Environment layer
is where these properties come together: it isolates the agent, it is fast, and
each run gives the same result.

<a id="environment-and-the-rest-of-the-theory"></a>

## Environment and the other concepts

- **[Tool](../primitives/tool.md)**: Tools are the interface to the Environment.
  A specific tool gives access to a small part of the Environment. A generative
  tool gives access to a large part.
- **[Guardrail](../harness/guardrail.md)**: Guardrails apply limits at the tool
  boundary. The Environment applies limits at the boundary of the resources. The
  two layers operate together. The Guardrails give policies at the tool level,
  and the Environment holds the agent at the resource level.
- **[Agent](../harness/agent.md)**: An Agent connects a Session to an
  Environment through a frontend. Two Agents with the same Session and different
  Environments are different Agents.
- **[Workspace](../patterns/workspace.md)**: A Workspace is a set of tools with
  a tree of names on the filesystem of an Environment. The Workspace is the view
  of the agent. The Environment is the layer below the Workspace.
- **[Continuity](../patterns/continuity.md)**: The `bootstrap` tool of
  Continuity is a setup step for the Environment. At the start of a Session, it
  changes the Environment from a cold condition to a condition for work.

## Related

- [Tool](../primitives/tool.md): the interface that the agent uses to operate on
  the Environment
- [Guardrail](../harness/guardrail.md): limits at the tool boundary that operate
  together with the limits of the Environment
- [Agent](../harness/agent.md): connects a Session to one Environment through a
  frontend
- [Workspace](../patterns/workspace.md): a set of tools with a tree of names on
  the filesystem of an Environment
