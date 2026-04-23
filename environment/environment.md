# Environment

Environment is the substrate the agent acts upon — the filesystem it reads, the processes it runs, the network it reaches, the credentials it holds. It is not part of the agent; it is the world the agent inhabits. Tools are the interface; Environment is what the interface reaches.

**Shape**: a resource bundle — code access, runtime, operating system, network reachability, credentials, installed software. Not a composition of primitives, harness, or patterns; a distinct concept in its own right.

## Why It Matters

An agent with the same [LLM](../primitives/llm.md), the same [Tools](../primitives/tool.md), the same [Session](../harness/session.md), and the same system prompt behaves very differently depending on what Environment it runs in. A coding agent on your laptop can do anything your shell can; the same agent in an ephemeral container can only touch what's inside the container; the same agent in a browser sandbox can only touch what the sandbox exposes.

This matters because the agent's _effective_ capability is a function of Environment, not of tool count. For [generative tools](../primitives/tool.md) like `bash`, this is explicit — `capability = f(environment)`. For specific tools, Environment is a passive configuration that bounds what the tool can reach. Either way, two agents with identical configuration but different Environments are different agents.

Environment is also where most of the security and reliability thinking lives. Guardrails enforce constraints at the tool boundary; Environment enforces them at the substrate boundary. Running an agent in a fresh container is how you bound blast radius when the agent has generative tools whose exact actions you can't predict.

## Formal Definition

```typescript
type Environment = {
  code?: CodeAccess; // git clone, local files, read-only mount
  runtime?: Runtime; // node, python, docker, databases
  os?: OSAccess; // shell, gh, ssh, kubectl
  network?: NetworkAccess; // what domains/ports are reachable
  credentials?: Credential[]; // API keys, tokens, SSH keys
  filesystem?: FilesystemAccess; // read/write paths
};
```

The fields are optional because Environments vary widely in what they include. A planning agent may need only `code` (read-only repo access). A full-stack coding agent needs `code + runtime + os`. A pure-reasoning agent may have an effectively empty Environment.

Environment is not a composition in the theory's sense — it doesn't build from primitives and patterns. It's the ontological substrate those primitives act on. Tools read and write Environment; [Guardrails](../harness/guardrail.md) constrain actions at the tool boundary before they reach Environment; sandboxing constrains Environment itself.

---

## Variants

Environments sit on a spectrum from "full host access" to "heavily restricted sandbox." Each point on the spectrum is a trade-off between agent capability and blast-radius control.

### Local OS

The agent runs as a process on the user's machine. Full access to the user's filesystem, installed tools, and credentials.

```typescript
const localOS: Environment = {
  code: localFS("~/projects/repo"),
  runtime: localNode,
  os: localShell,
  credentials: allUserCredentials, // agent has what the user has
};
```

**Seen in**: Claude Code CLI, Cursor, pi CLI, local coding agents.
**Strength**: Maximum capability, zero setup overhead, full tool ecosystem.
**Weakness**: No isolation — the agent can do anything the user can, including accidental damage to unrelated files.

### Container

Ephemeral isolated environment. The agent gets a fresh Linux system per task, with a specific image (tools, packages, OS version).

```typescript
const container: Environment = {
  code: gitClone("owner/repo"),
  runtime: dockerImage("node:20"),
  os: containerShell({ caps: ["NET_ADMIN"] }),
  credentials: [githubToken], // only what you explicitly inject
  network: { allow: ["github.com", "npmjs.org"] },
};
```

**Seen in**: OpenAI Codex, GitHub Copilot Workspace, CI-driven coding agents.
**Strength**: Isolation — agent can install packages, run servers, break things without affecting host. Clean state per task. Reproducible.
**Weakness**: Setup overhead. No persistence across tasks without explicit export. Slower cold start than local.

### Virtual Machine

Full OS isolation (not just container). Heavier but stronger isolation guarantees.

```typescript
const vm: Environment = {
  code: gitClone("owner/repo"),
  runtime: fullLinuxVM,
  os: vmShell,
  credentials: [scopedDeployKey],
};
```

**Seen in**: e2b, Modal, cloud-based long-running agents.
**Strength**: Stronger isolation than containers (kernel boundary). Can run anything a Linux host can.
**Weakness**: Provisioning latency. Higher cost per task. Usually overkill when containers suffice.

### Sandbox

A deliberately _restricted_ Environment — the agent is given less than it could technically have. Sandboxes exist on the restriction axis: each sandbox picks what to forbid.

```typescript
const codeSandbox: Environment = {
  code: readOnlyFS("/workspace"),
  runtime: pythonOnly,
  os: null, // no shell access at all
  network: { allow: [] }, // no network
  credentials: [], // no credentials
};

const browserSandbox: Environment = {
  // Agent can only act through a browser page. No filesystem, no shell.
  browser: chromiumInstance({ singlePage: true }),
  network: { allow: ["*"] },
};
```

**Seen in**: ChatGPT Code Interpreter (Python-only, no network, ephemeral), agent-browser sessions, enterprise document-analysis agents (read-only access to specific document tree).
**Strength**: Tightest blast-radius control. Suitable for untrusted inputs, third-party automation, compliance-bound tasks.
**Weakness**: Agent capability is capped at what the sandbox allows. For a generative-tool agent, this is often the right boundary; for a specific-tool agent it can be strictly weaker than tool-level guardrails.

Sandbox isn't a separate concept — it's a point on the Environment spectrum. The word "sandbox" signals intent (deliberate restriction for safety) but there's no sharp line between "sandboxed container" and "container with tight rules."

### Remote Host (SSH, kubectl, cloud APIs)

The agent's Environment is a remote system accessed over a protocol. The agent reaches in but isn't _inside_.

```typescript
const remote: Environment = {
  os: ssh({ host: "prod-server", key: deployKey }),
  runtime: kubectl({ cluster: "prod" }),
  credentials: [deployKey, kubeConfig],
};
```

**Seen in**: infrastructure agents, SRE automation, agents that drive remote systems.
**Strength**: Operate on production infrastructure without running there. Existing access controls (SSH keys, kube RBAC) apply.
**Weakness**: Network latency. Audit trails span systems. Session resumption is complex.

### API-Only / Network-Only

The agent has no filesystem or shell — only outbound API calls. Closest to "pure reasoning over external data."

```typescript
const apiOnly: Environment = {
  network: { allow: ["api.github.com", "api.slack.com"] },
  credentials: [githubToken, slackBotToken],
};
```

**Seen in**: Slack bots, workflow automation agents, simple integration agents.
**Strength**: Smallest attack surface. Stateless. Easy to reason about.
**Weakness**: Can't do anything the APIs don't expose. No local computation, no file handling.

---

## Environment and Generative Tools

The relationship between Tool and Environment changes with [tool expressiveness](../primitives/tool.md).

**Specific tools** (`file_read`, `db_query`, `gmail_send`): Environment is a passive configuration. The tool's schema bounds what the agent can reach. The agent cannot escape the tool set.

**Generative tools** (`bash`, code execution, generic HTTP): Environment becomes the capability boundary. The agent can install packages, spin up processes, reach new services — whatever the substrate allows.

```
Specific tools:    capability = Σ(tool schemas)
Generative tools:  capability = f(environment)
```

This is why Environment design matters most for agents with generative tools. A coding agent with `bash` in a fresh container has a radically different capability surface than the same agent with `bash` on the user's laptop. Sandboxing is how you bound that capability when you can't bound the tool.

---

## Environment and the Rest of the Theory

- **[Tool](../primitives/tool.md)**: Tools are the interface to Environment. Specific tools expose a narrow window; generative tools expose a wide one.
- **[Guardrail](../harness/guardrail.md)**: Guardrails enforce at the tool boundary; Environment enforces at the substrate boundary. Both layers compose — Guardrails are tool-level policy, Environment is substrate-level containment.
- **[Agent](../harness/agent.md)**: an Agent wires a Session to an Environment through a Frontend. Two Agents built from the same Session with different Environments are different Agents in practice.
- **[Workspace](../patterns/workspace.md)**: A Workspace is a hierarchical Tool-set _over_ an Environment's filesystem. Workspace is the agent-side abstraction; Environment is what the Workspace sits on.
- **[Continuity](../patterns/continuity.md)**: The `bootstrap` tool in Continuity is an Environment-setup step — it brings the Environment from cold state to working state at session start.

## Design Implications

**For agent builders**: Choose Environment deliberately. The default "my laptop, full access" is often wrong for anything beyond personal use. If the agent handles untrusted input, runs third-party code, or operates on production systems, the Environment is your primary blast-radius control — not the Guardrails.

**For security-conscious builders**: Treat Environment as the true capability boundary for generative-tool agents. Guardrails on `bash` catch the patterns you anticipated; the sandbox defines what the agent _could_ do even if it bypassed every Guardrail.

**For platform builders**: The Environment is increasingly a product. "Give agents a reliable, isolated Linux in one second" is the pitch of OpenAI Codex's cloud workspace, e2b, Modal, Cloudflare Workers for agents. The Environment layer is where isolation, speed, and reproducibility meet.

---

## Related

- [Tool](../primitives/tool.md) — the interface through which the agent reaches Environment
- [Guardrail](../harness/guardrail.md) — tool-boundary constraints that compose with Environment-boundary containment
- [Agent](../harness/agent.md) — wires a Session to a specific Environment via a Frontend
- [Workspace](../patterns/workspace.md) — hierarchical Tool-set over an Environment's filesystem
