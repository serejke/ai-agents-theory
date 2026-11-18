# Architecture

This document applies [First Principles](first-principles.md) to the
architecture of an agent system. It gives the sandbox tiers and the protection
of each tier. It gives the defenses with good research results, the defenses in
the products of companies and the defenses that attackers bypass. At the end, it
gives a list of checks for the architecture of an agent.

These names come from [First Principles](first-principles.md):

- **Process authority** is the set of operations that the operating system lets
  the process of the agent do.
- **Agent authority** is the set of operations that the agent can do in other
  services with its credentials, for example with a GitHub token.
- The **lethal trifecta** is the risk of an agent that can read private data,
  can read untrusted data and can send data out.

The most important facts of this document are these:

- A sandbox controls process authority. It does not control agent authority.
- The defenses with the strongest research results think of each output of the
  LLM as untrusted data. They put deterministic controls around the LLM.
- No company puts such a defense in its products. The defenses in products
  increase the work of the attacker, but they do not stop all attacks.
- The only known structural defense removes the lethal trifecta from the
  architecture.

## Sandbox tiers

A sandbox controls process authority: the files, the processes and the network
that the process of the agent can touch. Thus a sandbox is a limit of the
[Environment](../environment/environment.md), not a
[Guardrail](../harness/guardrail.md). The table compares the protection of each
tier with its start time. It also gives the attacks that each tier does not
stop.

| Tier                                                               | Start time                     | It stops                                                                                                                                                               | It does not stop                                                                                                                  |
| ------------------------------------------------------------------ | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Process limits (seccomp, AppArmor, Seatbelt)                       | Almost zero                    | Specified types of system calls, for example `ptrace`, raw sockets and mount namespaces                                                                                | Each operation through `openat()` or HTTP. These are almost all the important operations.                                         |
| Container (Docker, Podman, containerd)                             | 100 milliseconds to 2 seconds  | The containers keep their filesystems and process trees isolated. This is correct only without host mounts, host network and Docker socket.                            | An escape through the kernel, because all containers use the same kernel. `CVE-2024-21626` ("Leaky Vessels") is the best example. |
| gVisor                                                             | Approximately 1 second         | The protection of a container, and a filter of system calls in user space                                                                                              | System calls are 2 to 10 times slower. gVisor and the host use some parts of the same kernel.                                     |
| microVM (Firecracker, Cloud Hypervisor, Kata Containers)           | Approximately 125 milliseconds | The protection of gVisor, and a different kernel in each microVM. A kernel escape stays in the microVM.                                                                | Authority leaks, prompt injection and all the attacks on agent authority                                                          |
| Language sandbox (Pyodide and WASM, V8 isolates, Deno permissions) | 5 to 50 milliseconds           | By default, the runtime has no filesystem and no network.                                                                                                              | The runtime has a small set of libraries. Many agent tasks use the tools of the operating system.                                 |
| Temporary cloud sandbox (e2b, Modal, Daytona, Vercel Sandbox)      | 200 milliseconds to 2 seconds  | A product with one microVM for each task. The microVM can stop and then continue. The customer pays only for the time that it uses. By default, each task is isolated. | The same as a microVM: no protection of agent authority                                                                           |

### The defaults of 2026

In 2026, these decisions became the default:

- **One microVM for each task** is the standard method for agents that run the
  code of many customers on the same servers. These incidents caused this
  method:
  - In 2023, attackers showed escapes from the sandbox of ChatGPT Code
    Interpreter. Then the companies with containers for many customers changed
    to a virtual machine for each Session.
  - `CVE-2024-21626` made microVMs the default where a customer can be an
    attacker.
  - In 2025, e2b, Vercel Sandbox and Cloudflare Sandbox SDK became products.
    Thus microVMs became easy to use, and the customers pay a small quantity for
    them.

  Each microVM uses approximately 50 to 200 MB of memory and starts in
  approximately 125 milliseconds. These resources are satisfactory for agent
  tasks.

- **One sandbox for each MCP server** became the minimum after many supply chain
  incidents in 2025 and at the start of 2026. The MCP protocol does not control
  the capabilities of a server. The operator controls them on the host: each
  server runs in a different isolated Environment.

### A sandbox does not control agent authority

A microVM does not know the authority of the agent. In July 2025, an agent of
Replit erased a production database. The sandbox contained the process of the
agent fully. But the agent had the credential of the database, and the sandbox
let the dangerous query go through to the database.

The sandbox controlled only the access of the process to the disk. It did not
control if the agent can send this query with these credentials. A Guardrail
controls this, not the Environment. See
[References: Replit](references.md#replit-production-db-deletion).

<a id="the-defense-families-that-have-evidence"></a>

## Defenses with research results

Defenses are in two groups:

- **Probabilistic defenses** operate in the model. Examples are adversarial
  training, instruction hierarchy training, classifiers adjacent to the model
  and output filters. They decrease the success rate of each attack, but they do
  not remove the risk. Adaptive attacks bypass them.
- **Structural defenses** operate around the model. They think of each output of
  the model as untrusted data. Deterministic mechanisms do the control. Their
  protection stays correct also if the model does not obey.

All the important papers of 2025 use the same rule. Each output of the LLM is
untrusted data, and the deterministic controls are around the LLM. Only this
group gives guarantees. In 2025, most research moved to this group. The next
sections give the examples of this group.

<a id="dual-llm-origin-pattern"></a>

### The dual LLM pattern

In April 2023, Simon Willison gave the dual LLM pattern. CaMeL comes from this
pattern. The pattern has these parts:

- A privileged LLM has the tools, but it does not read untrusted data directly.
- A quarantined LLM reads the untrusted data, but it has no tools. It gives only
  symbols that the privileged LLM cannot read, for example `$VAR1` and `$VAR2`.
- A deterministic step, not an LLM, changes each symbol into its value.

The dual LLM pattern without other defenses fails on some tasks. In these tasks,
the privileged LLM must do reasoning about the untrusted data, for example "if
the email is angry, escalate". CaMeL is the better version for production. In
CaMeL, the privileged LLM gives code, not English. A deterministic interpreter
runs this code as a data flow program with types. This program replaces the
symbols.

No company uses only the dual LLM pattern in production. The pattern continues
in CaMeL, in the "tool filter" defense of AgentDojo and in the source and sink
model of OpenAI. See
[References: dual LLM pattern](references.md#dual-llm-pattern).

<a id="capability-tagged-plan-execution-camel-family"></a>

### CaMeL: a plan with capability labels

CaMeL is the strongest published defense and a clear example of a structural
defense. Debenedetti, Tramèr, Carlini, Shumailov and other authors at DeepMind
published it in March 2025. It operates in these steps:

1. A privileged LLM reads only the request of the user. This request contains no
   untrusted data. The LLM writes a program in a small language that is almost
   the same as Python. The program can only call approved tools and move data
   between values.
2. An interpreter runs the program. The interpreter is not an LLM. Untrusted
   data moves through the interpreter as values that the privileged LLM does not
   read.
3. Some steps must change natural language. For example, a step gets an email
   address from an email, or it writes a text in a different language. For these
   steps, the interpreter calls a quarantined LLM. The quarantined LLM has no
   tools. Its output comes back as values, not as changes to the program.
4. Each value has capability labels: a `readers` set for confidentiality and a
   `sources` set for integrity. Deterministic policy checks of these labels
   accept or reject each tool call.

**The guarantee.** CaMeL stops each attack that uses instructions in the data to
change the control flow or to add tool calls. The system completes the plan
before it reads untrusted data. Thus an instruction from a prompt injection
cannot become a tool call. The guarantee stays correct also if prompt injection
fully controls the two LLMs. The privileged LLM does not read the data of the
attacker, and the quarantined LLM has no tool authority.

**The effect on tasks.** On the AgentDojo benchmark, an agent with CaMeL
completes 77% of the tasks. An agent without a defense completes 84%. Thus the
difference is approximately 7 percent points. Each step that touches data uses
two model calls, and the interpreter also uses time. For guarantees that no
probabilistic defense gives, this effect is satisfactory.

**The limit.** CaMeL does not stop exfiltration through the arguments of a call
that the privileged LLM accepted. For example, the privileged LLM writes "send
email to extracted_address". An attacker can then make the quarantined LLM give
the email address of the attacker. The integrity labels must reject this value.

The author of the policy controls if they reject it correctly. Thus capability
systems change the problem. Without them, the model must reject the attack. With
them, the engineer must write a correct policy, and that problem is much better.

See [References: CaMeL](references.md#camel).

<a id="information-flow-control-fides-family"></a>

### Fides: information flow control

Fides uses the same structure as CaMeL, and it gives a more accurate description
of it. Costa, Köpf and other authors at Microsoft Research published it in
May 2025. Its planner follows confidentiality labels and integrity labels on
each value. It applies a security policy with deterministic checks. It also adds
operations that keep selected data away from the planners that it calls.

The paper tells clearly which flows information flow control at runtime cannot
find. These are leaks of meaning, and flows through the decisions of the
program, not through values. This limit also helps, because it shows the correct
limits of the guarantee. See [References: Fides](references.md#fides).

Siddiqui and other authors published "Permissive Information-Flow Analysis" in
October 2024. It gives a solution for the usual problem of information flow
control: too much taint. Basic information flow control puts a taint on each
output that an input with a taint touched. Then each workflow that gets external
data gives only untrusted data.

This method moves only the labels of the inputs that had an effect on the
output. It measures this effect with the attention of the model or with k-NN.
The paper shows a better result than the baseline in more than 85% of the tests.
See [References: Siddiqui](references.md#permissive-ifc).

NeuroTaint (Wang and other authors, 2026) shows a limit of standard information
flow control. Standard Bell-LaPadula information flow control with symbols gets
an F1 score of approximately 0.522 on their TaintBench. A method that also
follows changes of meaning, the effect of each input and memory across Sessions
gets 0.928. The result for an engineer: standard labels are necessary, but they
do not show all the flow of data in an LLM. Information flow control that knows
the meaning of the data is the possible next step. See
[References: NeuroTaint](references.md#neurotaint).

<a id="plan-then-execute-and-the-design-patterns-catalog"></a>

### The design patterns paper

In June 2025, Beurer-Kellner, Tramèr, Debenedetti and other authors published
"Design Patterns for Securing LLM Agents Against Prompt Injections". It is
almost a manual of these defenses. Each design pattern of the paper removes one
type of prompt injection attack with its structure:

- Action-Selector
- Plan-Then-Execute
- LLM Map-Reduce
- the dual LLM pattern
- Code-Then-Execute, with CaMeL as an example
- Context-Minimization

For each design pattern, the paper gives the attack that it removes. It also
gives the tasks where the design pattern is satisfactory and the tasks where it
is not.

All these design patterns obey one rule: make the plan first. The agent makes
its plan of tool calls before it reads untrusted data. A program that is not an
LLM runs the plan. Data comes back only as values, not as new instructions.
CaMeL is the strongest example of this rule. An engineer can also build a weaker
example without the full capability mechanism:

1. A planner LLM writes a plan of tool calls in a format that a program can
   read.
2. A program that is not an LLM runs the plan.
3. The outputs come back as values.

A plan first does not stop data leaks through arguments. For example, examine
the plan `for each email do summarize(email); maybe_send(reply)`. A prompt
injection in one email can change the reply. A plan first is necessary, but it
is not sufficient. Add capability labels to the arguments to stop the leaks
through the data flow. See
[References: the design patterns paper](references.md#design-patterns-paper).

<a id="capability-and-object-capability-frameworks-for-agents"></a>

### Capability frameworks for agents

The next step for agents is security that uses capabilities. MiniScope
(December 2025) uses capabilities to control the tools of an agent. Each task of
an agent gets a part of the capabilities of the agent that starts it. A
mechanism cuts this part, not an instruction in the prompt. See
[References: MiniScope](references.md#miniscope).

AgentGuardian (2026) tries to find access control policies in the operations of
the agent. Be careful with this method. An attacker can control the agent during
this process. Then the policy accepts the operations of the attacker. See
[References: AgentGuardian](references.md#agentguardian).

Permission as a tool is in production, but only as a basic version. Examples are
the permission prompts of Claude Code and the approval prompts of Cursor. No
company has a tool call API where the call contains a capability. In such an
API, only the agent that gives the capability can make it. This agent can also
decrease the authority of the capability. This is an open problem of research,
and MiniScope points to it.

In the words of the theory, this is [Topology](../patterns/topology.md) and a
[Guardrail](../harness/guardrail.md):

- The capability set is a structure of the Topology. It tells the authorities of
  each subagent and of each tool.
- A Guardrail policy applies the capability set at the point where the harness
  sends each tool call. It rejects each call if the caller does not have the
  capability for it.

<a id="what-vendors-ship-in-production"></a>

## The defenses in products

The defenses in products are different from the defenses of research. Each large
company has a stack of defenses. No company puts a structural defense with a
guarantee, for example CaMeL or Fides, in its products. The defenses in products
increase the work of the attacker, but they do not remove the risk.

**Anthropic and Claude Code:**

- Training with a constitution that makes the model reject dangerous requests.
- The user gets a prompt for the permission of each tool. The user can accept a
  tool one time or always.
- A list of commands that the agent cannot run.
- By default, the agent can only read files.
- An isolated virtual machine for each Session in cloud builds.
- Auto Mode: a check on the server examines tool outputs for prompt injection.
  It examines them before they go into the context of the agent.

**OpenAI, Atlas and Codex:**

- Instruction hierarchy training (Wallace and other authors, 2024). The model
  obeys the system first, then the user, then images and other input that is not
  text, then the output of tools.
- Watch Mode. On sensitive web pages, it stops the agent if the user does not
  look at it.
- A red team model that OpenAI made with reinforcement learning. It does attacks
  with social engineering.
- Instructions for users: examine each pair of a source and a sink.
- The public position of OpenAI: the problem stays open.

**Google, Gemini and Mariner:**

- Classifiers on the inputs.
- An instruction to "stay focused" at the start and at the end of the data.
- Training with feedback from persons on sets of prompt injection examples.
- Controls in Google Workspace.
- The paper of DeepMind about the defense of Gemini (May 2025) tells clearly
  that adaptive attacks find a path through the defenses again.

**Microsoft Copilot:**

- The XPIA classifier finds indirect prompt injection. It is on by default, and
  the user cannot deactivate it.
- Prompt Shields in the Azure AI Content Safety layer.
- Hooks for Defender.
- EchoLeak (`CVE-2025-32711`) bypassed this stack in production.

**The MCP protocol:**

- An OAuth 2.1 framework.
- Instructions for the connection between the client and the server.
- It recommends "user consent for tool invocation".
- The protocol gives almost all the security work to the client. Reports show
  tool poisoning and rug pulls, and the protocol does not stop them.

See [References: Anthropic](references.md#anthropic-claude-code-defense-stack),
[OpenAI](references.md#openai-hardening-atlas),
[DeepMind](references.md#deepmind-gemini-lessons-blog) and
[Microsoft](references.md#microsoft-xpia-defense).

### The defenses in products are layers of heuristics

The defenses in products are heuristics in layers. No company puts a structural
guarantee in its products. An attacker can bypass each layer of each stack
independently. The companies think that the attacker must bypass many layers.
This increases the work of the attacker, but it does not remove the risk.

### EchoLeak bypassed the largest stack

Microsoft has the largest stack of these companies. It has a dedicated XPIA
classifier, removal of links and the Azure AI Content Safety layer. One email
from an attacker bypassed this stack with zero clicks. The email used these
methods:

- Markdown links in the reference format
- images that the client fetches automatically
- a proxy of Teams that the Content Security Policy accepts

Thus an attacker bypassed a defense with a classifier in production, in M365, a
very large product. If a company tells you "we use a classifier", the classifier
decreases the risk. It is not a structural defense. See
[References: EchoLeak](references.md#echoleak).

<a id="negative-results-worth-knowing"></a>

## Defenses that fail

It is important to know the defenses that fail, and also the defenses that
operate correctly. Many persons use these defenses, and they look correct, but
attackers can bypass them:

- **Spotlighting** (Hines and other authors, Microsoft, March 2024) puts marks
  on untrusted data. It adds marks in the data, changes the data to a different
  code, or puts marks at the start and at the end. On standard prompts, the
  success rate of attacks falls below 2%. Then adaptive attacks bypass it. Use
  it as one layer of heuristics, not as the structural defense.
- **Training to reject prompt injection does not operate against new attacks.**
  Zhan and other authors (Findings of NAACL 2025) examined eight defenses.
  Adaptive attacks bypassed all eight, and the success rate stayed above 50% for
  each defense. Pasquini ("The Attacker Moves Second", October 2025) makes the
  point stronger: an attacker who knows a defense that does not change bypasses
  it.
- **An attacker can use constrained decoding for an attack.** DictAttack (Zhang
  and other authors, March 2025) puts the payload in the grammar, which controls
  the output. The prompt, which holds the data, stays safe. The success rate of
  the attack is 94% to 99.5% on GPT-5, Gemini 2.5 Pro, DeepSeek-R1 and
  GPT-OSS-120B. A JSON format for tool call output does not control the meaning
  of the output.
- **Unicode characters bypass small classifier models.** Homoglyphs, characters
  with zero width and characters that a person cannot see bypass these
  classifiers. Reports through 2025 show this clearly.
- **EchoLeak bypassed the dedicated XPIA classifier of Microsoft.** This attack
  occurred in production, in a very large product. The result for the
  architecture: an attacker can bypass each defense with a classifier.

See [References: defenses that fail](references.md#negative-results).

<a id="a-working-checklist-for-an-agent-design"></a>

## Checks for an agent architecture

This list of checks gives the position of an engineer who makes the architecture
of agents in 2026. The position must stay correct after a careful check. It is
not a full manual. It is a set of decisions. If each decision has a good
solution, the agent has a strong security position.

### Apply the lethal trifecta filter to the architecture

For each code path of the agent, find the parts of the lethal trifecta that it
touches. These parts are private data, untrusted data and external
communication. If one code path touches all these parts, an attacker can use it
for exfiltration. Remove one part from the architecture. Examples:

- An email agent that only reads and does not send has no external
  communication.
- A coding agent on a public repository has no private data.
- A research agent reads web pages, but its privileged LLM operates on a context
  with only clean data. No untrusted data gets to this LLM.

### Use a sandbox, and know its limit

A sandbox is necessary, but it is not sufficient. By default, run code in a
microVM for each task. Run each MCP server in a different sandbox. A sandbox
controls only process authority. It does not control agent authority. The
incidents of Replit and EchoLeak occurred in sandboxes.

### Use capability labels where the integrity of data flow is important

CaMeL is the strongest published method, but no company puts it in its products.
You can build the weaker version from
[the design patterns paper](#plan-then-execute-and-the-design-patterns-catalog).
Its planner LLM writes a plan of tool calls, and a program that is not an LLM
runs it. The outputs come back as values, not as instructions. This stops the
largest type of indirect prompt injection: external data that changes the
control flow.

### Think of MCP servers, skills and plugins as a dangerous supply chain

Pin the version of each of them, and examine them. Do the same checks as for the
dependencies of your code. Also scan their contents for payloads in natural
language. Put each server in a sandbox on the host. Do not run skills from an
unknown source with the default authority of the agent. For each task, give them
the minimum authority for that task.

### Add probabilistic defenses, but do not use only these defenses

Think that the LLM and the classifier can give incorrect results. Adversarial
training, checks for prompt injection, instruction hierarchy training and output
filters all decrease the risk. None of them is the structural defense. Use them
together with the structural defenses of this document. Do not use only one of
them.

### Compare the rate of each attack with the number of attacks

A defense that decreases the success rate of each attack from 50% to 1% is a
large step. But it is not sufficient when the attacks are many. An agent can
read more than a thousand documents with untrusted data in 24 hours. For this
agent, the important number is the total risk in the 24 hours, not the rate of
each attack. Make your plan for the unusual attack that gets through, not for
the usual turn.

### Think of the output as an important attack surface

The user thinks that the reply of the agent is safe. Thus the reply can contain
a payload. If untrusted data in the context controls the output, these items can
contain an attack:

- Markdown images
- URLs that the client fetches automatically
- the text of links
- code blocks that look safe

The program that shows the output to the user is a part of the security
boundary. By default, remove the URLs that the client fetches automatically.
Show the output only after a Guardrail examines it.

### Control the damage with Topology, not with the model

Use [Topology](../patterns/topology.md) to make the authority of each agent
small. Then if an attacker controls one agent, the attacker can do only a small
quantity of damage. Use these methods:

- Divide the work between subagents.
- Give less authority to each subagent than to the agent that starts it.
- Keep the agents that read data isolated from the agents that change data or
  send data.

Do not give one agent all the parts of the lethal trifecta. Do not think that
its training will reject the attack.

### The architecture in 2026

In 2026, a correct architecture has these layers:

- data flow with capability labels
- a plan before the tool calls
- a classifier as an auxiliary layer
- a sandbox as a second auxiliary layer

Model defenses and classifier defenses are only auxiliary layers for agent
authority. They cannot be the structural defense. The structural defense removes
the lethal trifecta from the architecture. No other structural defense is known.

## Related

- [First Principles](first-principles.md): the first principles of the decisions
  of this document.
- [References](references.md): the papers, the documents of companies and the
  incidents that this document refers to.
- [Environment](../environment/environment.md): the sandbox tiers as variants of
  the Environment.
- [Guardrail](../harness/guardrail.md): the part of the harness where capability
  policies operate together with the structural defenses.
- [Topology](../patterns/topology.md): the Topology for security. Examples are
  the dual LLM pattern, subagents with less authority and a planner apart from
  the agent that does the plan.
