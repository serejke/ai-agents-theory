# First Principles

Agent security has two different problems, and they are not connected. The first
problem is process authority: the operations that the operating system lets the
agent process do. The second problem is agent authority: the operations that the
agent can do in other services with the credentials that you gave it. A defense
for one problem does not stop the attacks of the other problem. Most engineers
and most texts of companies about their products do not keep the two problems
apart. This error causes almost each incorrect decision about agent security.

<a id="the-two-axes"></a>

## Two different problems

### Process authority

Process authority is the set of operations that the operating system lets the
agent process do. With too much process authority, an agent can do these
operations:

- Run `rm -rf /`.
- Fill or erase the filesystem of the host.
- Use a large quantity of cloud resources in a loop that does not stop.
- Start new processes until the computer stops.
- Send `~/.ssh/id_rsa` to an attacker through the shell.

The tools of the operating system that isolate processes are the solution to
this problem: containers, virtual machines, microVMs and language sandboxes.
Engineers made these tools for this problem.
[Environment](../environment/environment.md) gives these variants.

### Agent authority

Agent authority is the set of operations that the agent can do in other services
with its credentials. For example, an agent has your GitHub token, because its
task is to change your repository:

1. An attacker writes an instruction in the text of a GitHub issue.
2. The agent reads the issue and obeys the instruction.
3. The agent opens a pull request that contains a backdoor.

The pull request goes out through a usual `git push` with a correct signature,
from the sandbox of the agent. **No process boundary stops this.** The authority
goes across the boundary because the agent cannot do its task without it.

### The two problems are not connected

Process authority can cause damage without agent authority, for example in a
loop that does not stop. Agent authority can cause damage in a sandbox that
isolates the process fully. An example is the Replit incident (2025-07): an
agent erased a database that customers used. See
["Replit production DB deletion"](references.md#replit-production-db-deletion).
A defense for one problem does not help with the other problem.

Process authority is the easier problem, and engineers know a good solution for
it. [Architecture](architecture.md) gives the levels of sandboxes for it. Thus,
after this section, this document examines only agent authority.

<a id="the-lethal-trifecta"></a>

## The lethal trifecta

The lethal trifecta of Simon Willison is the best description of the risk of
agent authority. An agent has the lethal trifecta when it has all of these
properties at the same time:

1. **Access to private data.** The agent can read the data that the attacker
   wants: your email, your files, your credentials or internal documents.
2. **Access to untrusted data.** The agent reads input that the attacker
   controls. Examples are a web page, the text of an email, a GitHub issue and a
   Jira task. Other examples are a PDF, the output of an MCP tool and a RAG
   document.
3. **A path for data to go out.** The agent can make output that goes to a
   location on the network. Examples are an HTTP request, an email and a message
   in a channel. Other examples are a file in a location that other persons can
   read, and a link that a user interface shows.

If an agent has all three properties, exfiltration is possible. Then the only
safe defense is structural: the architecture of the agent must remove one
property. Do not think that the model will not obey the attacker. Without one
property, the agent cannot do the full attack. With all three properties, only
the model is a limit to the attack. Tests show that the model does not stop all
attacks, thus it cannot be that limit.

The lethal trifecta is a problem of information flow control. In information
flow control, untrusted data must not change the data that goes to a private
sink. When you remove one property of the lethal trifecta, you cut the path
between the source and the sink. You can do this before the system runs. That is
better than to think that the model will stop the attack when it runs.

See ["The lethal trifecta"](references.md#the-lethal-trifecta).

<a id="what-sandboxes-actually-solve"></a>

## The problem that sandboxes stop

Sandboxes are the solution to the problem of process authority. They set the
parts of the operating system that the agent process has access to. Examples are
containers, microVMs, language sandboxes, gVisor and temporary cloud sandboxes.
Each agent that runs code must have a sandbox.

Sandboxes are not the solution to the problem of agent authority. These facts
show it.

### Authority goes across the boundary

A coding agent in a sandbox that pushes to GitHub must keep the GitHub token in
the sandbox. An email agent in a sandbox must keep its IMAP credentials. Thus
the sandbox boundary lets through the authority that the agent must have. It
also lets through each operation that an attacker can make the agent do with
this authority. The correct output has the same bytes as the output of an
attack. Examples of this output are a `git push`, an SMTP message and an API
call with a signature.

### A confused deputy attack occurs in each sandbox

In a confused deputy attack, the agent operates for the user, but a different
person controls the agent. That person controls some data that the agent reads.
The first example in a system that customers used was "Living off Microsoft
Copilot" by Bargury and other authors, at Black Hat USA 2024:

1. An attacker gave documents to a usual user.
2. These documents controlled the Copilot of the user.
3. The Copilot sent private data of the organization to the attacker.

The agent and the user had correct access to the data. The attacker did not have
access, but the attacker controlled the exfiltration. The sandbox has no effect
here, because the agent has correct access. See
["Living off Microsoft Copilot"](references.md#living-off-microsoft-copilot).

### The output of the model is the attack

The reply of the model is the payload. The user thinks that the output of the
agent is correct, and the attacker controls the output. Examples:

- A Markdown image has a URL of the attacker. The data goes to the attacker in
  the query string of the URL.
- A short description of code does not show the dangerous lines of a diff.
- A link that looks safe goes to a web page that tells you to write your
  password.

No process sandbox stops an attacker who changes the output.

### Two different problems

Sandboxes are the solution to the first of these problems, but not to the
second:

- "can the agent's process do bad things in the host?"
- "can the agent be tricked into doing bad things using authority I gave it?"

The most frequent error is to use a sandbox as the solution to the second
problem.

<a id="why-prompt-injection-is-architecturally-unsolved-at-the-model-layer"></a>

## The model cannot stop prompt injection

The model layer cannot fully stop prompt injection, because the input of an LLM
has no boundary between instructions and data. This is the most important fact
of agent security.

### SQL has a boundary between instructions and data

Queries with parameters stop SQL injection. A prepared query has a boundary
between the instruction and the data, and each part has its type. The
instruction is the text of the SQL query, and the data is the parameters. The
database applies that boundary when it parses the query. The boundary is
structural. Thus the parser cannot mix the instruction and the data.

### An LLM has no boundary between instructions and data

The input of an LLM is one sequence of tokens, and all the tokens are of the
same type. The LLM uses one operation to obey an instruction and to find if a
token is an instruction or data. No syntax, type system or runtime mechanism
shows which tokens have more authority. Only the patterns that the model got
from its training show this.

Thus a defense in the model can make the risk small, but it cannot make the risk
zero. Training against attacks can decrease the risk of each attack from more
than 50 percent to less than 10 percent. For a risk of zero, a parser must
identify the instructions and the data. The architectures of the models of this
time do not have this mechanism.

### The companies that make the models agree

The companies that make the models give the same result:

- OpenAI told in public that prompt injection is "unlikely to ever be fully
  solved" with the model architectures of this time. Its head of information
  security wrote this in 2025-10, when OpenAI made the Atlas browser safer.
- The Opus 4.5 system card of Anthropic calls prompt injection the primary risk
  for agents that no company can fully remove.
- The CaMeL paper of DeepMind (2025-03) is the most careful published defense
  with architecture. The paper wants to give "defeating prompt injections by
  design", because defenses from training are not sufficient.

### The risk increases with each attack

The risk of one attack is not the correct value to examine. Attack tests on
Claude Opus 4.5 (2025-11) show how the risk increases:

| Strong attacks | Risk that one of the attacks gets its result |
| -------------- | -------------------------------------------- |
| 1              | 4 to 5 percent                               |
| 10             | approximately 33 percent                     |
| 100            | approximately 63 percent                     |

The strongest models of other companies get a risk of 90 percent in the same
conditions. An agent that operates without a person can read more than 1000
documents with untrusted data in 24 hours. For this agent, the correct value to
examine is the total risk of all the attacks. If the risk of one attack is not
very small, an attacker will almost always control the agent in some hours. A
defense that decreases the risk from 50 percent to 1 percent is a good result,
but it is not sufficient.

See [Attack tests](references.md#frontier-model-red-team-results),
[AgentDojo](references.md#agentdojo) and [AgentHarm](references.md#agentharm).

<a id="indirect-injection-is-the-operational-threat"></a>

## Indirect prompt injection is the primary risk

Prompt injection has two types, and they are not equally dangerous. For an agent
that operates without a person, indirect prompt injection is the primary risk.

### Direct prompt injection

In direct prompt injection, the user is the attacker. The user writes dangerous
instructions to go around the safety rules of the model. This is a jailbreak. It
causes problems, but large damage from it is not frequent. The damage is equal
to the permissions of the user. In most agent systems, an attack of the user on
the user is not an important threat model.

### Indirect prompt injection

In indirect prompt injection, the attacker controls data that the agent reads as
part of its task. Microsoft calls it "XPIA" ("Cross-Prompt Injection Attempt").
The user is not the attacker. The attacker is a different person who wrote some
text that the agent reads:

- an email or a web page
- a GitHub issue, a Jira task or a Slack message
- a RAG document, a tool description or a part of a URL.

The agent reads this text into the same context window where it selects its tool
calls. The text contains instructions for the agent.

### The causes of the risk

Indirect prompt injection is the primary risk for these causes:

- **It gives the authority of the user to the attacker.** The user has tools and
  credentials. The attacker has no tools and no credentials. Indirect prompt
  injection runs the instructions of the attacker with the authority of the
  user.
- **The attack surface is all the data that the agent reads.** A coding agent
  reads the scripts of `package.json`, the `README` files of dependencies,
  GitHub issues, CI logs and the configuration of branches. A browser agent
  reads each page that it opens. An email assistant reads each email that it
  gets. No source of these is a known source of instructions.
- **The user does not do an operation.** EchoLeak was a security problem of
  Microsoft 365 Copilot (2025-06, `CVE-2025-32711`). An attacker sent one email
  with special text. The user did not open a link, but the private data of the
  organization went to the attacker. See [EchoLeak](references.md#echoleak).
- **It connects with each other function of the agent.** Indirect prompt
  injection adds to tools, web pages, [Memory](../patterns/memory.md) and supply
  chain attacks. One sentence in one source of untrusted data can go through all
  the functions of the agent.

In 2025, NIST and OWASP put indirect prompt injection first in their lists of
risks for LLM programs.

<a id="the-supply-chain-compounding"></a>

## The supply chain

Each dependency of an agent is a path for prompt injection. Examples are a
package, an MCP server, a skill, a plugin, a part of the system prompt and a RAG
document.

### The payload is natural language

In usual software, a supply chain attack sends dangerous bytes that run as code.
The defenses are data about the source of each package, signatures, lockfiles
and builds that give the same result each time. In an agent, the payload is
natural language, and it runs in the context window of the model. A signature on
a `SKILL.md` file does not tell you if its instructions are good. A lockfile
does not stop an MCP server that changes its tool descriptions after you install
it, and does not tell you about the change.

The usual defenses are necessary. They make sure that the bytes of each
dependency are correct. But they do not stop a payload in natural language. Thus
the supply chain of an agent system must have two types of defense:

- The usual defenses of the software supply chain.
- A check of the text of each dependency for prompt injection.

### MCP servers

By default, MCP has no security model for servers. A user installs a program
that keeps credentials and sends tool calls to other services. MCP does not make
servers use signatures, sandboxes or limits on their authority. Many CVEs from
the second half of 2025 and from the start of 2026 show that MCP has a small
number of security controls:

- the RCE in MCP Inspector
- the RCE in LibreChat
- the RCE pattern in the STDIO connection of MCP.

MCP has no defense against two known types of attack:

- **Tool poisoning.** A server writes dangerous instructions in its tool
  descriptions, and the LLM reads them.
- **Rug pull.** A server changes its tool definitions after the user accepts the
  server.

See ["MCP architectural RCE"](references.md#mcp-architectural-rce) and
["Tool poisoning attacks"](references.md#tool-poisoning-attacks).

### Skill marketplaces

Examples of skill marketplaces are Claude Code skills, Cursor rules and OpenAI
custom GPTs. Hermes Agent can also "fetch skill from arbitrary GitHub repo". In
2025, Snyk examined skills that users published, and found prompt injection in
36 percent of them. In 2026-02, persons found more than 1000 dangerous skills in
one skill marketplace.

### Typosquatting

Typosquatting of npm and pip packages is also an attack on agents. In 2026-02,
the `SANDWORM_MODE` attack published more than 19 dangerous npm packages:

1. The packages used names that are almost the same as the names of tools for
   developers and AI tools.
2. When a developer installed a package, the package started an MCP server.
3. The MCP server used indirect prompt injection against the AI assistant of the
   developer.
4. The assistant sent SSH keys, cloud credentials and npm tokens to the
   attacker.

This attack is indirect prompt injection in a usual supply chain attack. The
dangerous package does not get more permissions. It only gives instructions to
the agent.

<a id="what-this-implies-for-the-theory"></a>

## Security in the theory

Each part of agent security agrees with a concept of the theory:

- A sandbox is a variant of [Environment](../environment/environment.md). The
  levels of sandboxes are the local operating system, the container, the microVM
  and the language sandbox. They are variants of Environment, not a new concept.
  [Architecture](architecture.md) shows them as variants.
- [Guardrail](../harness/guardrail.md) is the point where the harness stops tool
  calls. A deterministic policy can stop a tool call if its arguments come from
  untrusted data. It can also stop the tool call until the user accepts it.
  These policies are Guardrail policies in the meaning of the theory.
- [Topology](../patterns/topology.md) contains the known security structures.
  They are not new tiers:
  - a Session that makes a plan and a different Session that does the plan
  - the dual LLM pattern
  - subagents with a smaller set of tools and permissions.

From these facts, two results follow.

### Sandboxes and Guardrails compose

A sandbox and a Guardrail compose, and one does not replace the other. A sandbox
in the Environment sets a limit to the authority of the process. A Guardrail
sets a limit to agent authority, because it stops tool calls. The two are
necessary, and one of them without the other is not sufficient. Most systems
that customers use put too much work into the sandbox and not sufficient work
into the Guardrail.

### The Guardrail controls the data in tool arguments

CaMeL and Fides stop a tool call when the label of its arguments shows data that
is not safe. That operation is a Guardrail policy that uses information flow
control. The policy is in the `pre` hook of the harness. The policy is "deny if
arguments derive from untrusted sources and tool sink is high-confidentiality".
The theory has the structural mechanism. Only the policy is new.

The field moves to these architectures:

- data with labels that tell which tools can use it
- a plan before the first tool call
- the dual LLM pattern.

These architectures are compositions of harness, Environment and pattern
concepts. The risks of this document cause them. They do not add tiers to the
theory. They add better Guardrail policies and clear limits in the Topology.

## Related

- [Architecture](architecture.md): the levels of sandboxes and the types of
  defense in the architecture.
- [References](references.md): the papers, company documents, CVEs and incidents
  of this document.
- [Environment](../environment/environment.md),
  [Guardrail](../harness/guardrail.md) and [Topology](../patterns/topology.md):
  the concepts that this part of the book applies to security.
