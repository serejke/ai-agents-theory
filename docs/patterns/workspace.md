# Workspace

A Workspace is a structured, persistent data space where agents read input, write intermediate artifacts, produce output, and collaborate with other agents and humans. It is the answer to "where does the agent work?" — a question that individual primitives don't address.

**Composition**: [Tool](../primitives/tool.md) (filesystem operations) + [Channel](../primitives/channel.md) (shared directories between agents) + [ContextProvider](context.md) (hierarchy as navigable context). Optional: permissions, versioning, collaboration.

## Why It Matters

An agent needs somewhere to work — not just tools to call and memory to recall, but a **place** where it reads context, writes artifacts, and collaborates. [Channel](../primitives/channel.md) answers "how does agent A's output reach agent B?" [Memory](../primitives/memory.md) answers "what does the agent know from past sessions?" Workspace answers **"where does the agent read, think, and write?"**

Claude Code operates on a local filesystem. Devin gets a workspace with files and a shell. Enterprise agents need access to company documents with permissions and versioning. Coding agents navigate repository trees. Research agents write notes to directories that other agents read. In every case, the agent's effectiveness depends on the structure and semantics of the data space it inhabits.

## Formal Definition

```typescript
type Workspace = {
  read: (path: string) => Content;
  write: (path: string, content: Content) => void;
  list: (path: string, pattern?: string) => Entry[];
  search: (query: string) => Result[];

  // What distinguishes Workspace from raw filesystem access:
  permissions?: PermissionLayer; // identity-based access control
  versioning?: VersioningLayer; // automatic history of changes
  structure: HierarchicalNamespace; // directory tree = knowledge graph
};
```

---

## What Makes Workspace Distinct

| Property     | Channel                           | Memory                   | **Workspace**                                     |
| ------------ | --------------------------------- | ------------------------ | ------------------------------------------------- |
| Persistence  | Run-scoped                        | Cross-session            | Task or project-scoped                            |
| Structure    | Flat keys or conventions          | Flat entries             | **Hierarchical** (tree = knowledge graph)         |
| Access       | Agent-to-agent                    | Agent-to-self (recall)   | **Multi-party** (agents + humans as peers)        |
| Permissions  | None                              | None                     | **Identity-based** (inherited from org)           |
| Versioning   | None                              | None or manual           | **Automatic** (every write is tracked)            |
| Content type | Intermediate results              | Extracted knowledge      | **Artifacts** (documents, analyses, drafts, data) |
| Primary use  | Pass data between pipeline stages | Remember across sessions | **Do work** — read context, produce output        |

---

## The Filesystem Convergence

Every major agent system converges on filesystem semantics for the workspace interface:

| System                                 | Workspace                               |
| -------------------------------------- | --------------------------------------- |
| Claude Code                            | Local repository filesystem             |
| Cursor                                 | Repository + editor buffer              |
| Devin                                  | Sandboxed workspace with files + shell  |
| OpenAI Codex                           | Sandboxed cloud workspace               |
| GitHub Copilot Workspace               | Virtual repository checkout             |
| Enterprise platforms (Box, SharePoint) | Virtual filesystem over managed storage |

The interface is always the same four operations: `read`, `write`, `list`, `search`. This convergence isn't accidental — it follows from a property of the underlying model.

---

## Cognitive Affinity — Why Filesystem Wins

Not all tool interfaces are equal from the LLM's perspective. The degree to which an LLM can reliably reason about and use a tool depends on how well the interface maps to patterns in its training data. Call this **cognitive affinity**.

Filesystem operations have maximum cognitive affinity because:

1. **Training saturation.** LLMs are trained on massive codebases where file manipulation is the dominant pattern. `read_file`, `write_file`, `ls`, `mkdir`, `grep` are among the most frequently occurring operations in code. The model has seen millions of examples of correct filesystem reasoning.

2. **Simple CRUD semantics.** Four operations cover the entire interface. No complex state machines, no multi-step protocols, no session management. Each operation is independent and idempotent (reading doesn't change state, writing is a complete replacement).

3. **Composable identifiers.** Paths are hierarchical, human-readable, and composable: `/legal/contracts/vendors/acme_2024.pdf`. The agent can reason about path structure — parent directories imply category, sibling files imply related documents, depth implies specificity.

4. **Human alignment.** Humans already organize information in hierarchies. The filesystem metaphor matches how people think about document organization, which means the agent's navigation mirrors human expectations.

This has a practical design implication: **when building tools for agents, translate complex domain interfaces into filesystem-like metaphors** where possible. An enterprise storage API with folder IDs, metadata endpoints, and permission queries becomes:

```
Complex API:                          Filesystem facade:
GET /folders/:id/items          →     list("/contracts/vendors/")
GET /files/:id/content          →     read("/contracts/vendors/acme.pdf")
POST /files + POST /metadata    →     write("/analysis/acme_risks.md")
```

The agent doesn't need to understand the enterprise API. It understands files and directories. The translation layer handles the rest. This is the [Tool boundary](../primitives/tool.md) — but implemented specifically as a **cognitive facade**: an interface designed not for human developers but for LLM reasoning.

---

## Structural Navigation — Hierarchy as Retrieval

Agent systems typically retrieve context through search (see [Memory retrieval strategies](../primitives/memory.md)). Workspace enables an additional strategy: **structural navigation** — the agent traverses a hierarchy to find relevant context.

```typescript
// The agent navigates the tree, using directory names as semantic cues
list("/")
  → company/, personal/, projects/

list("/company/")
  → legal/, finance/, engineering/, hr/

list("/company/legal/contracts/")
  → vendors/, clients/, employment/

read("/company/legal/contracts/vendors/acme_2024.pdf")
```

At each level, the directory name tells the agent whether to go deeper or look elsewhere. The tree structure **is** a knowledge graph — hierarchy encodes categorical relationships, proximity encodes relatedness, naming encodes semantics.

### Structural navigation vs. semantic search

| Property                       | Semantic search (RAG)                                 | Structural navigation                   |
| ------------------------------ | ----------------------------------------------------- | --------------------------------------- |
| Requires preprocessing         | Yes (embedding, indexing)                             | No                                      |
| Preserves hierarchy            | No (flattened into vectors)                           | Yes                                     |
| Handles organizational context | Poorly (loses "this contract is under legal/vendors") | Naturally                               |
| Works with new/unseen content  | After re-indexing                                     | Immediately                             |
| Agent effort per query         | One search call                                       | Multiple list/read calls                |
| Best for                       | "Find anything relevant to X"                         | "Navigate to the right area, then read" |

The two strategies complement each other. Semantic search excels at cross-cutting queries ("find all mentions of GDPR compliance"). Structural navigation excels at scoped exploration ("what contracts do we have with this vendor?"). A well-designed workspace supports both.

---

## Permission Models

Agent systems need access control. The question is where to enforce it. Two architecturally distinct approaches:

### Agent-level guardrails

Permissions enforced at the orchestration layer, wrapping tool execution:

```typescript
// From the Guardrail primitive
const noFinanceAccess: Guardrail = {
  pre: (action) => {
    if (action.params.path?.startsWith("/finance/"))
      return { deny: "agent not authorized for finance data" };
    return { allow: true };
  },
};
```

The agent sees the full workspace structure but gets blocked when it tries to access restricted areas. Permissions are configured per-agent, maintained by whoever deploys the agent.

### Data-level permissions

Permissions enforced by the workspace itself, based on the identity of the caller:

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

The agent never sees files it can't access — they're invisible in `list()` results. Permissions are inherited from existing organizational policies (LDAP, SSO, org hierarchy), not re-implemented per agent.

### When to use which

| Scenario                        | Approach               | Reason                                                                                  |
| ------------------------------- | ---------------------- | --------------------------------------------------------------------------------------- |
| Personal agent on local files   | Agent-level guardrails | No org policies to inherit; simple deny rules suffice                                   |
| Enterprise agent on shared data | Data-level permissions | Must respect existing ACLs; can't re-implement org policies per agent                   |
| Multi-tenant agent platform     | Both                   | Data-level for tenant isolation; agent-level for per-agent restrictions within a tenant |

The enterprise case reveals why permission-aware workspaces matter: an organization with 10,000 employees and complex access policies can't re-encode those policies as agent [Guardrails](../primitives/guardrail.md). The workspace inherits them from the data layer. The agent is just another identity in the permission system — like a new employee who gets access based on their role, not a custom access list.

---

## Workspace Variants

### 1. Local Filesystem

The simplest workspace. Agent operates directly on the OS filesystem.

```typescript
const localWorkspace: Workspace = {
  read: (path) => fs.readFile(path),
  write: (path, content) => fs.writeFile(path, content),
  list: (path) => fs.readdir(path),
  search: (query) => grep(query, "."),
};
```

**Seen in**: Claude Code, Cursor, terminal-based agents.
**Strength**: Zero setup, full tool ecosystem (git, npm, compilers).
**Weakness**: No versioning (unless git-backed), no permissions beyond OS user, no collaboration without external tools.

### 2. Git-Backed Workspace

Local filesystem with git providing versioning, branching, and collaboration.

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

**Seen in**: Claude Code (implicitly — operates in git repos), GitHub Copilot Workspace, any code-focused agent.
**Strength**: Full version history, branching for isolation, human review via PRs.
**Weakness**: Designed for code, not arbitrary documents. Git's merge model assumes line-based text.

### 3. Sandboxed Workspace

Isolated environment (container, VM, cloud instance) with a fresh filesystem per task.

```typescript
const sandboxedWorkspace: Workspace = {
  read: (path) => container.exec(`cat ${path}`),
  write: (path, content) => container.writeFile(path, content),
  list: (path) => container.exec(`ls ${path}`),
  search: (query) => container.exec(`grep -r "${query}" .`),
};
```

**Seen in**: Devin, OpenAI Codex, cloud-based coding agents.
**Strength**: Full isolation — agent can install packages, run servers, break things without affecting host. Clean state per task.
**Weakness**: Setup overhead, no persistence across tasks without explicit export.

### 4. Virtual / Managed Workspace

A facade over enterprise storage. The agent sees filesystem semantics; the backend provides permissions, versioning, search, and collaboration.

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

The agent calls `read("/contracts/acme.pdf")`. The backend resolves the path to an enterprise storage object, checks permissions against the caller's identity, extracts text via OCR if needed, and returns content. The `write` creates a new version, not an overwrite. The `search` may use embeddings, full-text indexing, or both.

**Seen in**: Box's agent filesystem, SharePoint/OneDrive agent integrations, enterprise content platforms repositioning as agent workspaces.
**Strength**: Inherits enterprise permissions, automatic versioning, supports collaboration between agents and humans as peers.
**Weakness**: Complexity of the translation layer. Latency of remote operations. Dependency on the platform provider.

---

## Workspace as Collaboration Surface

A property that distinguishes Workspace from Channel: it naturally supports **multi-party collaboration** between agents and humans.

```
/project/
  requirements.md        ← human wrote this
  architecture.md        ← agent drafted, human revised
  /implementation/
    auth_module.ts       ← agent wrote
    auth_module_test.ts  ← agent wrote
  /reviews/
    security_review.md   ← second agent analyzed auth_module.ts
    human_feedback.md    ← human responded to security review
```

No special protocol needed. Both agents and humans read and write files. The workspace is the shared surface. This is fundamentally different from Channel (agent-to-agent only) and Memory (agent-to-self only).

In enterprise settings, agent-written artifacts appear in the same team folders as human-written documents. The agent becomes a participant in collaboration, not a separate system the human has to check. Versioning makes it safe — nothing is overwritten, everything is tracked, any change can be reviewed or reverted.

---

## Design Implications

**For agent builders**: Your agent needs a workspace, not just tools. Even if it's "just" a local filesystem, recognize it as the workspace and design accordingly — directory structure encodes context, naming conventions matter, workspace layout is part of the agent's cognitive environment.

**For platform builders**: The workspace is the strategic layer. Whoever provides the managed workspace for enterprise agents controls the context layer — and context is what makes agents useful. This is why enterprise content platforms (Box, SharePoint, Google Drive) are racing to reposition as agent workspaces: they already have the documents, the permissions, the versioning, and the collaboration infrastructure.

**For tool designers**: When building tools that expose data to agents, consider a filesystem facade over your domain. Complex APIs with IDs, nested resources, and pagination become `list` + `read` + `write` + `search` over a virtual directory tree. The translation costs you engineering effort once; it saves the LLM cognitive effort on every interaction.

---

## Related

- [Tool](../primitives/tool.md) — workspace operations are tools; workspace is a structured composition of them
- [Channel](../primitives/channel.md) — shared directories within a workspace serve as channels between agents
- [ContextProvider](context.md) — directory hierarchy is a navigable context source
- [Guardrail](../primitives/guardrail.md) — agent-level permission enforcement within a workspace
- [App Inversion](../app-inversion/architecture.md) — workspace is the agent-side surface of the tool boundary
