# First Principles

Agent security decomposes into two orthogonal axes. Most engineering and most vendor marketing conflate them; conflating them is the source of nearly every wrong design decision in the field.

## The Two Axes

**Process authority.** What the operating system lets the agent's process do. The agent runs `rm -rf /`, eats the host filesystem, runs up a cloud bill in a recursive loop, fork-bombs the machine, exfiltrates `~/.ssh/id_rsa` via shell redirection. This is a containment problem and is what classical OS isolation tools — containers, VMs, microVMs, language sandboxes — were designed to address. See the variants in [Environment](../environment/environment.md).

**Agent authority.** What the agent's identity is allowed to do in upstream services. The agent legitimately holds your GitHub token because that is its job. An attacker plants an instruction inside a GitHub issue body. The agent reads the issue, follows the instruction, opens a backdoored pull request. The egress is a normal, signed `git push` originating from inside whatever sandbox you put the agent in. **No process boundary blocks this** — the authority crosses the boundary by design, as a capability the agent must have to function.

The two axes are independent. Containment failure can occur with no agent-authority misuse (e.g., a runaway loop). Agent-authority misuse can occur inside a perfectly contained sandbox (e.g., the Replit production-database deletion of July 2025; see [References § Replit production DB deletion](references.md#replit-production-db-deletion)). A defense designed for one axis does not transfer to the other.

The remainder of this document is about agent authority. Process authority is well understood at the systems-engineering level — the [Architecture](architecture.md) document gives the operational ladder for sandbox tiers — but it is the easier of the two problems and the one the field already knows how to engineer.

## The Lethal Trifecta

The cleanest framing of agent-authority risk is Simon Willison's _lethal trifecta_: an agent has it when all three properties hold simultaneously.

1. **Access to private data** — the agent can read information the attacker wants to obtain (your inbox, files, credentials, internal documents).
2. **Exposure to untrusted content** — the agent reads input the attacker controls (a webpage, an email body, a GitHub issue, a Jira ticket, a PDF, the return value of an MCP tool, a RAG document).
3. **Ability to communicate externally** — the agent can produce output that reaches a network destination (HTTP request, email, message in a channel, file written to a shared location, link rendered in a UI).

If all three hold, exfiltration is achievable; the only reliable defense is _structural_ — sever one leg by architecture, not by hoping the model resists. An agent with any two of the three is bounded by the missing leg; an agent with all three is bounded only by model behavior, and model behavior is empirically not reliable enough to be the boundary.

This reduces to a non-interference question. The classical formulation is information-flow control: untrusted-source data must not influence high-confidentiality sinks. Cutting a leg of the trifecta is the same as cutting the path between source and sink — and you can do it at design time, by construction, rather than at runtime, by hoping.

See [References § The lethal trifecta](references.md#the-lethal-trifecta).

## What Sandboxes Actually Solve

Sandboxes — containers, microVMs, language sandboxes, gVisor, ephemeral cloud sandboxes — solve process authority. They constrain what the agent's _process_ can reach in the operating system. They are necessary infrastructure for any agent that executes code.

They do not solve agent authority. Three lines of evidence make this concrete.

**Authority crosses the boundary by construction.** A sandboxed coding agent that pushes to GitHub must hold the GitHub token _inside_ the sandbox. A sandboxed email agent must hold IMAP credentials. The sandbox boundary is permeable to exactly the capabilities the agent needs — and to whatever an attacker can steer the agent into doing with those capabilities. The legitimate egress (git push, SMTP, signed API call) is byte-identical to the malicious egress.

**Confused-deputy attacks are sandbox-invariant.** The agent acts on behalf of the user but is steered by a third party who controls some content the agent reads. The original demonstration in production was Bargury et al.'s "Living off Microsoft Copilot" (Black Hat USA 2024), which exfiltrated tenant data through a benign user's Copilot using documents shared by an attacker. The agent had legitimate access to the data; the user had legitimate access to the data; the attacker did not — and yet the attacker drove the egress. Sandbox is irrelevant; the agent has legitimate access. See [References § Living off Microsoft Copilot](references.md#living-off-microsoft-copilot).

**The output is the attack vector.** The model's reply is the payload. Markdown-image rendering with attacker-controlled URLs encodes data into query strings; misleading code summaries omit malicious diff lines; hyperlinks that look benign point to phishing endpoints. The user trusts the agent's output; the attacker controls what the agent says. No process sandbox addresses output manipulation.

The general statement: **sandboxes answer "can the agent's process do bad things in the host?" They do not answer "can the agent be tricked into doing bad things using authority I gave it?"** These are different questions. Treating sandbox isolation as the answer to the second is the modal misdiagnosis.

## Why Prompt Injection Is Architecturally Unsolved at the Model Layer

This is the load-bearing claim in the field. SQL injection was solved by parameterized queries — the prepared statement is a typed boundary between _instruction_ (the SQL skeleton) and _data_ (bound parameters), and the database engine enforces that boundary at parse time. The boundary is structural; the parser cannot confuse one for the other.

Large language models have no such boundary. The transformer's input is one undifferentiated token stream. The decision "is this token an instruction or data?" is made _by the same forward pass that follows the instruction_. There is no syntactic, type-system, or runtime mechanism to distinguish privileged from unprivileged tokens — only learned correlations.

As long as that holds, defense at the model layer is asymptotic, not absolute. Adversarial training can move per-attempt success rates from above fifty percent toward single-digit percent. It cannot reach zero. The mechanism that would make it reach zero — a parser-level distinction between instruction and data — does not exist in current architectures.

This conclusion is not contested by frontier labs. OpenAI's CISO publicly stated (October 2025, in announcing hardening for the Atlas browser) that prompt injection is "unlikely to ever be fully solved" with current model architectures. Anthropic's Opus 4.5 system card frames prompt injection as the central unsolved threat for agentic AI. DeepMind's CaMeL paper (March 2025) — the most rigorous published architectural defense — explicitly states its goal is "defeating prompt injections by design" _because training-based defenses are insufficient_. The labs that ship the models converge on the same answer.

The compounding problem at scale: per-attempt success rate is the wrong unit. Recent red-team data on Claude Opus 4.5 (November 2025): one strong attack succeeds with probability around four to five percent, ten attempts succeed with probability around thirty-three percent, one hundred attempts succeed with probability around sixty-three percent. Frontier models from other vendors hit ninety percent under similar conditions. For an autonomous agent processing thousands of untrusted documents per day, the relevant metric is the compounded probability — and it converges to certain compromise within hours at non-trivial per-attempt rates. A defense that takes the per-attempt rate from fifty percent to one percent is real progress and yet still not enough.

See [References § Frontier model red-team results](references.md#frontier-model-red-team-results), [§ AgentDojo](references.md#agentdojo), and [§ AgentHarm](references.md#agentharm).

## Indirect Injection Is the Operational Threat

Prompt injection has two flavors. They are not equally severe.

**Direct injection** — the user is the attacker, pasting hostile instructions to bypass refusals. This is jailbreaking. Annoying, rarely catastrophic; the blast radius equals the user's own privileges, and user-against-self is not an interesting threat model in most agent deployments.

**Indirect injection** (Microsoft's term: XPIA, Cross-Prompt Injection Attempt) — the attacker controls _content the agent reads as part of its task_. The user is benign. The attacker is some third party who wrote an email, a webpage, a GitHub issue, a Jira ticket, a Slack message, a RAG document, a tool description, a URL fragment. The agent loads attacker-controlled bytes into the same context window where it makes tool-calling decisions, and those bytes contain operational instructions.

For autonomous agents, indirect injection is the dominant threat. Reasons:

- **Privilege escalation primitive.** The user has tools and credentials. The attacker has neither. Indirect injection is the bridge that takes the attacker's instructions and runs them with the user's authority.
- **Attack surface is everything the agent reads.** A coding agent reads `package.json` scripts, dependency READMEs, GitHub issues, CI logs, branch configurations. A browser agent reads every DOM it lands on. An email assistant reads every inbound email. None of those sources are authenticated as instructions.
- **Zero user interaction.** The June 2025 EchoLeak vulnerability in Microsoft 365 Copilot (CVE-2025-32711) was a _zero-click_ exfiltration: an attacker sent a single crafted email; the victim never clicked anything; tenant data left the boundary. See [References § EchoLeak](references.md#echoleak).
- **Composability.** Indirect injection chains with everything: tool use, browsing, persistent memory, supply-chain compromises. One injected sentence in one untrusted source can pivot through the agent's whole capability surface.

NIST and OWASP both rank indirect prompt injection as the top LLM-application threat as of 2025.

## The Supply-Chain Compounding

Every dependency edge an agent has — package, MCP server, skill, plugin, system-prompt fragment, RAG document — is a _prompt-injection edge_.

This is the conceptual shift: classical software supply chain treats the threat as _hostile bytes executed as code_ and addresses it with provenance, signing, lockfiles, reproducibility. In an agent context, the payload is _natural language_, and the execution surface is the model's context window. Code-signing a `SKILL.md` does not tell you whether its instructions are honest. Lockfiles do not protect against an MCP server's tool descriptions being silently rewritten after install.

Specific vectors as of 2026:

- **MCP servers.** The Model Context Protocol's default trust model is none. A user installs a binary that holds credentials and proxies tool calls; the protocol does not require signing, sandboxing, or capability scoping. Multiple CVEs in the second half of 2025 and early 2026 (MCP Inspector RCE, LibreChat, the STDIO-transport architectural RCE pattern) reflect that protocol-level controls are minimal. Tool poisoning — malicious instructions embedded in the server's tool descriptions, which the LLM reads — and rug-pulls — a server changing tool definitions post-trust — are documented unfixed-at-protocol-level attack classes. See [References § MCP architectural RCE](references.md#mcp-architectural-rce) and [§ Tool poisoning attacks](references.md#tool-poisoning-attacks).
- **Agent skill marketplaces.** Claude Code skills, Cursor rules, OpenAI custom GPTs, Hermes-style "fetch skill from arbitrary GitHub repo." A 2025 Snyk audit reported that thirty-six percent of scanned community skills contained prompt injection. February 2026 saw over a thousand malicious skills surfaced in a single agent marketplace.
- **npm/pip-style typosquatting in agent contexts.** February 2026's SANDWORM_MODE campaign deployed nineteen-plus malicious npm packages impersonating dev and AI tools; on install they deployed an MCP server that used indirect prompt injection against the developer's AI assistant to harvest SSH keys, cloud credentials, and npm tokens. Indirect injection fused with classical supply chain: the malicious package does not need to escalate by itself — it whispers to the agent.

Provenance and signing are still necessary for the byte-level integrity of these dependencies. They do not address the natural-language payload. The supply-chain layer of an agent system needs _content auditing for prompt-injection payloads_ in addition to classical software-supply-chain controls.

## What This Implies for the Theory

The theory's existing concepts cover the substrate cleanly:

- [Environment](../environment/environment.md) is the place sandboxing goes. The sandbox tier ladder (local OS → container → microVM → language sandbox) is a refinement of Environment variants, not a new concept. The [Architecture](architecture.md) document treats it as such.
- [Guardrail](../harness/guardrail.md) is the place tool-call interception goes. Deterministic policies that refuse a tool call when its arguments derive from untrusted sources, or require user confirmation, are Guardrail policies in the formal sense.
- [Topology](../patterns/topology.md) is where planner/executor splits, dual-LLM, and capability-attenuated subagents go. These are not new tiers; they are recognized arrangements.

Two observations follow from putting it together:

**Sandboxing and Guardrails compose, they do not substitute.** Environment-level sandboxing bounds process authority. Guardrails bound agent authority by intercepting tool calls. Both are necessary; neither is sufficient alone. Most production stacks today over-invest in the first and under-invest in the second.

**Securing data flow into tool arguments is a Guardrail responsibility.** When the architecture defenses in CaMeL or Fides refuse a tool call because its arguments carry an unacceptable integrity label, that is a Guardrail policy informed by information-flow tracking. The hook point is the harness's `pre` interception; the policy is "deny if arguments derive from untrusted sources and tool sink is high-confidentiality." The structural mechanism already exists in the theory; what is novel is the policy.

The architectural moves the field is converging on — capability-tagged dataflow, plan-first execution, dual-LLM patterns — are compositions of existing harness, environment, and pattern concepts driven by the threat model in this document. They do not require new tiers; they require richer Guardrail policies and explicit Topology constraints.

## Related

- [Architecture](architecture.md) — the operational ladder of sandbox tiers and the architectural defense families.
- [References](references.md) — papers, vendor publications, CVEs, and incidents cited above.
- [Environment](../environment/environment.md), [Guardrail](../harness/guardrail.md), [Topology](../patterns/topology.md) — the existing concepts this branch extends with security-specific applications.
