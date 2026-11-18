# Workspace

A Workspace is the area where an agent does its work: a tree of files and
folders that stays after each step. The agent reads its input there and writes
its work files and its output. Other agents and persons read and write the same
files. For example, Claude Code operates in the folder of a repository.

**Composition**: [Tool](../primitives/tool.md) (filesystem operations) +
[Channel](channel.md) (directories that two or more agents use) +
[PromptLoading](prompt-loading.md) (the tree as a context source that the agent
moves through). Optional parts: permissions, a version history, access for
persons and other agents.

<a id="why-it-matters"></a>

## Value

An agent must have an area of work, not only tools to call and a memory to read.
In this area, it reads context, writes artifacts and operates together with
other agents and persons. The primitives do not give this area. Workspace is the
**pattern** that gives it. Each of these patterns has a different function:

- A [Channel](channel.md) moves the output of one agent to a different agent.
- [Memory](memory.md) keeps the knowledge of an agent from previous Sessions.
- A Workspace is the area where the agent reads, thinks and writes.

Each agent system has a different area of work:

- Claude Code operates on a local filesystem.
- Codex gets a Workspace in a sandbox on a server of OpenAI.
- An agent of a company must read the documents of the company, with permissions
  and a version history.
- Coding agents move through the tree of a repository.
- Research agents write their results to directories, and other agents read
  these results.

In each example, the structure and the meaning of this data area control the
quality of the work of the agent.

<a id="formal-definition"></a>

## Definition

```typescript
// Pseudocode. An implementation selects its own text, binary and
// directory-entry types.
type Workspace = {
  read: (path: string) => string | Buffer;
  write: (path: string, content: string | Buffer) => void;
  list: (
    path: string,
    pattern?: string,
  ) => Array<{ name: string; kind: "file" | "dir" }>;
  search: (query: string) => Array<{ path: string; snippet: string }>;

  // These fields make a Workspace more than raw filesystem access:
  permissions?: (identity: string, path: string) => boolean; // access control from the caller's identity
  versioning?: { history: (path: string) => Array<{ id: string; at: Date }> }; // automatic history of changes
  structure: "hierarchical"; // the directory tree is a knowledge graph
};
```

<a id="what-makes-workspace-distinct"></a>

## How a Workspace is different from a Channel and Memory

A Workspace keeps the artifacts of a task or a project in a tree. Agents and
persons use it together, with permissions and an automatic version history. A
[Channel](channel.md) and [Memory](memory.md) do not have these properties.

| Property    | Channel                            | Memory                       | **Workspace**                                            |
| ----------- | ---------------------------------- | ---------------------------- | -------------------------------------------------------- |
| Time        | One run                            | Across Sessions              | One task or one project                                  |
| Structure   | Keys without a tree, or name rules | Entries without a tree       | **Tree** (a knowledge graph)                             |
| Access      | Agent to agent                     | Agent to the same agent      | **Agents and persons as equal users**                    |
| Permissions | None                               | None                         | **From the user** (rules of the company)                 |
| Versions    | None                               | None, or manual              | **Automatic** (for each write)                           |
| Data        | Results between steps              | Knowledge from previous work | **Artifacts** (documents, reports, first versions, data) |
| Function    | Move data between pipeline steps   | Keep knowledge for Sessions  | **Do work**: read context, make output                   |

<a id="the-filesystem-convergence"></a>

## All agent systems use a file interface

All large agent systems use the operations of a filesystem as the interface of
their Workspace:

| System                   | Workspace                                 |
| ------------------------ | ----------------------------------------- |
| Claude Code              | Local filesystem of the repository        |
| Cursor                   | Repository and the open buffers of Cursor |
| Codex                    | Workspace in a sandbox on a server        |
| GitHub Copilot Workspace | Copy of the repository                    |
| Box and SharePoint       | File facade on the storage of a company   |

The interface always has the same four operations: `read`, `write`, `list` and
`search`. This is not an accident. It comes from a property of the LLM.

<a id="cognitive-affinity--why-filesystem-wins"></a>

## Files agree with the training of the LLM

An LLM uses a tool correctly more frequently when the interface of the tool
agrees with the patterns of its training data. This book calls this property
**cognitive affinity**. Filesystem operations have the highest cognitive
affinity, for these causes:

1. **Training data.** The training data of an LLM has very large quantities of
   code, and operations on files occur most frequently in this code.
   `read_file`, `write_file`, `ls`, `mkdir` and `grep` are in the most frequent
   operations in code. The model has seen more than a million examples of
   correct reasoning about files.
2. **A small set of operations.** Four operations do all the work of the
   interface. The interface has no state machine, no protocol of many steps and
   no connection to keep. Each operation operates independently of the other
   operations. `read` does not change the data. `write` replaces all the file.
3. **Paths that the agent can read.** A path is a tree that persons can read,
   and the agent can make new paths from it, for example
   `/legal/contracts/vendors/acme_2024.pdf`. The agent can think about the
   structure of a path. The folder above a file gives its type. Files in the
   same folder are related documents. A deeper path identifies a smaller group
   of documents.
4. **The structure that persons use.** Persons put their information in trees.
   The filesystem agrees with how persons think about their documents. Thus the
   agent and a person move through the files with the same method.

### Give the agent a file facade

Cognitive affinity gives a rule for the tools of agents. **When you make tools
for agents, give each large API a file facade where possible.** An API of
company storage with folder IDs, metadata endpoints and permission queries
becomes this facade:

```
Complex API:                          Filesystem facade:
GET /folders/:id/items          →     list("/contracts/vendors/")
GET /files/:id/content          →     read("/contracts/vendors/acme.pdf")
POST /files + POST /metadata    →     write("/analysis/acme_risks.md")
```

The agent can operate without knowledge of the API of the company. It knows
files and directories, and the facade does all the other work. This is a special
type of [Tool boundary](../primitives/tool.md): a **cognitive facade**. A
cognitive facade is an interface for the reasoning of the LLM, not for a
developer.

<a id="workspace-variants"></a>

## Workspace variants

The variants are different in the location of the files and in the properties
that the backend adds.

| Variant           | Examples                                                                 | Good points                                                                                                                               | Bad points                                                                                                                                      |
| ----------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Local filesystem  | Claude Code, Cursor, agents in a terminal                                | It has no configuration. All the usual tools operate, for example git, npm and compilers.                                                 | It has no versions without git. It has no permissions other than the user of the operating system. Work with other persons uses external tools. |
| Git               | Claude Code in git repositories, GitHub Copilot Workspace, coding agents | It keeps the full version history. Branches isolate the work. Persons examine the changes in pull requests.                               | Git is for code, not for all types of documents. The merge model of git operates on lines of text.                                              |
| Sandbox           | Codex, coding agents on servers                                          | It isolates the agent. The agent can install packages, run servers and break the system, and the host stays safe. Each task starts clean. | The configuration of each sandbox is slow. No data stays after the task if the harness does not export it.                                      |
| Managed workspace | Box, SharePoint and OneDrive for agents, document platforms of companies | It gets the permissions of the company and makes versions automatically. Agents and persons operate together as equal users.              | The facade is not easy to make. Operations on the network have latency. The company cannot operate without the platform provider.               |

<a id="1-local-filesystem"></a>

### Local filesystem

The agent operates directly on the filesystem of the operating system. This
variant has the smallest number of parts.

```typescript
const localWorkspace: Workspace = {
  read: (path) => fs.readFile(path),
  write: (path, content) => fs.writeFile(path, content),
  list: (path) => fs.readdir(path),
  search: (query) => grep(query, "."),
};
```

<a id="2-git-backed-workspace"></a>

### Git workspace

A git workspace is a local filesystem in which git gives versions, branches and
access for other persons. Claude Code uses this variant without a special
configuration, because it operates in git repositories.

```typescript
const gitWorkspace: Workspace = {
  read: (path) => fs.readFile(path),
  write: (path, content) => {
    fs.writeFile(path, content);
    gitAdd(path);
  },
  list: (path) => fs.readdir(path),
  search: (query) => grep(query, "."),
  versioning: {
    history: (path) => gitLog(path),
    revert: (path, version) => gitCheckout(version, path),
    branch: (name) => gitBranch(name),
  },
};
```

<a id="3-sandboxed-workspace"></a>

### Sandbox workspace

A sandbox gives each task a new filesystem. The sandbox can be a container, a
virtual machine or a server. A sandbox is a variant of the
[Environment](../environment/environment.md).

```typescript
const sandboxedWorkspace: Workspace = {
  read: (path) => container.exec(`cat ${path}`),
  write: (path, content) => container.writeFile(path, content),
  list: (path) => container.exec(`ls ${path}`),
  search: (query) => container.exec(`grep -r "${query}" .`),
};
```

<a id="4-virtual--managed-workspace"></a>

### Managed workspace

A managed workspace is a file facade on the storage of a company. The agent sees
a filesystem. The backend gives permissions, versions, semantic search and
access for persons.

```typescript
const managedWorkspace: Workspace = {
  read: (path) => enterpriseStorage.getContent(resolvePath(path)),
  write: (path, content) =>
    enterpriseStorage.createVersion(resolvePath(path), content),
  list: (path) => enterpriseStorage.listChildren(resolvePath(path)),
  search: (query) => enterpriseStorage.semanticSearch(query),
  permissions: enterpriseACL,
  versioning: automaticVersioning,
};
```

When the agent calls `read("/contracts/acme.pdf")`, the backend does these
steps:

1. It finds the storage object of the path.
2. It compares the permissions with the user of the caller.
3. If the document is a scan, it changes the image into text.
4. It gives the text to the agent.

A `write` makes a new version and keeps the previous version. A `search` can use
embeddings, a text index or the two.

<a id="structural-navigation--hierarchy-as-retrieval"></a>

## Find context with the tree

A Workspace lets an agent find context with **structural navigation**. The agent
moves through the tree, and the names of the folders tell it where to go. Agent
systems usually find context with queries (see [Memory](memory.md)). Structural
navigation is an added method.

```typescript
// The agent moves through the tree. The directory names tell it where to go.
list("/")
  → company/, personal/, projects/

list("/company/")
  → legal/, finance/, engineering/, hr/

list("/company/legal/contracts/")
  → vendors/, clients/, employment/

read("/company/legal/contracts/vendors/acme_2024.pdf")
```

At each level, the name of the directory tells the agent to go deeper or to go
to a different area. The tree **is** a knowledge graph:

- The levels of the tree give the groups of documents.
- Items in the same area of the tree are related.
- The names give the meaning.

<a id="structural-navigation-vs-semantic-search"></a>

### Structural navigation and semantic search

The two methods help each other. Semantic search is best for queries across all
the tree, for example "find all mentions of GDPR compliance". Structural
navigation is best in one area, for example "what contracts do we have with this
vendor?". A good Workspace lets the agent use the two methods.

| Property                    | Semantic search (RAG)                                     | Structural navigation               |
| --------------------------- | --------------------------------------------------------- | ----------------------------------- |
| Work before the first query | Yes: embeddings and an index                              | No                                  |
| Keeps the tree              | No: embeddings do not keep the tree                       | Yes                                 |
| Context of the organization | Bad: it does not keep "this contract is in legal/vendors" | Good, with no added work            |
| New documents               | After a new index                                         | Immediately                         |
| Agent calls for each query  | One `search` call                                         | Two or more `list` and `read` calls |
| Best use                    | "Find anything relevant to X"                             | "Go to the correct area, then read" |

<a id="workspace-as-collaboration-surface"></a>

## Work with persons and other agents

A Workspace lets agents and persons operate together on the same files. A
[Channel](channel.md) cannot do this, because only agents use a Channel.

```
/project/
  requirements.md        ← a person wrote this
  architecture.md        ← an agent wrote it, a person changed it
  /implementation/
    auth_module.ts       ← an agent wrote this
    auth_module_test.ts  ← an agent wrote this
  /reviews/
    security_review.md   ← a second agent examined auth_module.ts
    human_feedback.md    ← a person answered the security review
```

This work has no special protocol. Agents and persons read and write files, and
the Workspace is the surface that they all use. A Channel connects only agents
to agents. [Memory](memory.md) connects an agent only to the same agent.

In a company, the artifacts of agents are in the same folders as the documents
of persons. The agent is a part of the work. It is not a different system that a
person must check. Versions make this safe. No `write` removes data, and the
Workspace records each change. A person can examine each change and put back the
previous version.

<a id="permission-models"></a>

## Permissions

An agent system can apply permissions at two different points: in the harness of
the agent, or in the Workspace. The two layers can operate together.

<a id="agent-level-guardrails"></a>

### Permissions in the agent

The harness applies the permissions around each tool call, with a
[Guardrail](../harness/guardrail.md) policy:

```typescript
// A Guardrail policy on the tool-dispatch hooks
const noFinanceAccess: Guardrail = {
  pre: (action) => {
    if (action.params.path?.startsWith("/finance/"))
      return { deny: "agent not authorized for finance data" };
    return { allow: true };
  },
};
```

The agent sees all the structure of the Workspace. The Guardrail stops the agent
when it tries to go into an area without permission. Each agent has a different
set of permissions, and the person who deploys the agent keeps them.

<a id="data-level-permissions"></a>

### Permissions in the data

The Workspace applies the permissions from the user who calls it:

```typescript
const managedWorkspace: Workspace = {
  read: (path) => {
    const identity = getCurrentIdentity();
    if (!acl.canRead(identity, path)) throw new AccessDenied(path);
    return storage.read(path);
  },
  list: (path) => {
    const identity = getCurrentIdentity();
    return storage.list(path).filter((entry) => acl.canRead(identity, entry));
  },
};
```

The agent does not see a file that it cannot read, because `list()` does not
show it. The permissions come from the policies of the organization, for example
LDAP, SSO or the structure of the organization. No person writes them again for
each agent.

<a id="when-to-use-which"></a>

### Which method to use

| Agent                            | Method                   | Cause                                                                                                                      |
| -------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Agent of one person, local files | Permissions in the agent | The person has no policies of an organization. Rules that stop some operations are sufficient.                             |
| Agent of a company, company data | Permissions in the data  | The agent must obey the access lists of the company. No person can write the company policies again for each agent.        |
| Platform for many customers      | The two methods          | Permissions in the data isolate the customers. Permissions in the agent decrease the access of each agent of one customer. |

The company example shows the cause that a Workspace with permissions is
important. An organization with 10,000 persons and many access policies cannot
write these policies again as [Guardrails](../harness/guardrail.md) of each
agent. The Workspace gets the policies from the data layer. The agent is one
more user in the permission system. It is the same as a new person in the
company, who gets access from their function, not from a special list.

<a id="design-implications"></a>

## Decisions for builders

**For agent builders**: Your agent must have a Workspace, not only tools. A
local filesystem is also a Workspace. Give it a good structure. The directory
structure gives context, and the names are important. The layout of the
Workspace is a part of the context of the agent.

**For platform builders**: The Workspace is the most important layer for a
platform company. The company that supplies the managed workspace for company
agents controls the context layer. Context is the cause that agents do good
work. Thus document platforms of companies, for example Box, SharePoint and
Google Drive, move quickly to become Workspaces for agents. They have the
documents, the permissions, the versions and the systems for work together.

**For tool builders**: When you make tools that give data to agents, think about
a file facade on your data. APIs with IDs, resources in resources and pages of
results become `list`, `read`, `write` and `search` on a directory tree. You do
the work of the facade one time. It makes the work of the LLM easier at each
call.

## Related

- [Tool](../primitives/tool.md): Workspace operations are Tools. A Workspace is
  a composition of these Tools with a structure.
- [Channel](channel.md): the directories that two or more agents use in a
  Workspace are Channels between these agents.
- The [PromptLoading](prompt-loading.md) pattern uses the directory tree as a
  context source that the agent can move through.
- [Guardrail](../harness/guardrail.md): it applies permissions in the agent, in
  a Workspace.
