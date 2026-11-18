# Security

This part of the book examines agent security. It tells the bad results that an
agent can cause, and how the concepts of the theory give defenses against them.
It also connects the words of the theory to the research and to the products of
companies.

Security is not a new tier. The area of attack is in these concepts of the
theory: [Environment](../environment/environment.md),
[Guardrail](../harness/guardrail.md), [Tool](../primitives/tool.md) and
[Topology](../patterns/topology.md). This part of the book tells the limits that
these concepts must apply before an agent is safe for customers.

## Documents

- **[First Principles](first-principles.md)**: the basic model of agent
  security.
  - Agent security has two different problems. A sandbox stops the bad
    operations of the agent process. A sandbox does not stop the bad operations
    that the agent does with its authority.
  - The lethal trifecta.
  - The cause that the model layer cannot stop prompt injection with the
    architectures of the models of this time.
  - The supply chain makes the risk larger: MCP servers and skill marketplaces.
- **[Architecture](architecture.md)**: the decisions of the person who makes the
  structure of an agent.
  - The levels of sandboxes, and the safety that each level gives.
  - The types of defense that research shows to be good. One type gives labels
    to data that tell which tools can use the data, as in CaMeL. The other types
    are information flow control and a plan before the agent runs tools.
  - The defenses that large companies use in their products, and the defenses
    that they only examine in research.
  - A list of checks for the structure of an agent in 2026.
- **[References](references.md)**: each paper, company document, CVE and
  incident that the other two documents give, with a short description of each
  item. Use it to find the source of a fact. Use it also to find the correct
  technical result of a paper, not only its short public description.

<a id="connection-to-the-taxonomy"></a>

## Connection to the theory

Agent security uses these concepts of the theory directly:

- **[Environment](../environment/environment.md).** A sandbox is a limit at the
  level of the Environment. Its variants are the local operating system, the
  container, the microVM and the language sandbox. Each variant gives the agent
  access to a different part of the system.
- **[Guardrail](../harness/guardrail.md).** The Guardrail is the part of the
  harness that stops tool calls. At this point, a deterministic policy can stop
  a tool call or remove private data from it. The policy can also stop the tool
  call until the user accepts it. Most of the defenses in this part of the book
  are Guardrails. They compose with structural limits on the data that can go
  into each tool argument.
- **[Topology](../patterns/topology.md).** These structures are Topology
  decisions:
  - a Session that makes a plan and a different Session that does the plan
  - the dual LLM pattern
  - subagents with a smaller set of tools and permissions.

  Some Topologies are for security, and other Topologies are for performance.

The three documents of the list contain the full text. This file only shows the
documents and their connections.
