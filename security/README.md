# Security

This branch of the book is a thematic deep-dive into agent security: what bad outcomes an agent can produce, what the existing taxonomy says about defending against them, and where the theory's vocabulary connects to current research and shipped products.

It is not a new tier. The structural concepts — [Environment](../environment/environment.md), [Guardrail](../harness/guardrail.md), [Tool](../primitives/tool.md), [Topology](../patterns/topology.md) — already cover the surface; this branch zooms in on what those concepts must enforce to make a working agent safe to deploy.

## Documents

- **[First Principles](first-principles.md)** — the conceptual frame. Agent security decomposes into two orthogonal axes: misused process authority (sandboxes solve it) and misused agent authority (sandboxes do not solve it). The "lethal trifecta" framing. Why prompt injection cannot be solved at the model layer with current architectures. The supply-chain compounding (MCP servers, skill marketplaces).
- **[Architecture](architecture.md)** — the architect's stance. Sandbox tier ladder with what each layer buys you. The defense families that actually have evidence (CaMeL-class capability execution, information-flow control, plan-then-execute). What major vendors actually ship in production vs what they research. A working checklist for a 2026 agent design.
- **[References](references.md)** — every paper, vendor publication, CVE, and documented incident cited from the first two documents, with per-entry summaries. Read this when you want the source for a claim, or when you want to see the actual mechanical contribution of a paper rather than its marketing summary.

## Connection to the Taxonomy

Security thinking touches three existing parts of the theory directly:

- **[Environment](../environment/environment.md)** — sandboxing is an Environment-level constraint. The variants (local OS, container, microVM, language sandbox) carve up the spectrum of what the agent can reach.
- **[Guardrail](../harness/guardrail.md)** — tool-call interception is the harness layer at which deterministic policy can refuse, redact, or require confirmation. The architectural defenses in this branch are largely Guardrails composed with structural constraints on what data may flow into what tool argument.
- **[Topology](../patterns/topology.md)** — the planner/executor split, dual-LLM, and capability-attenuated subagent patterns are topology choices. Security-driven Topologies exist alongside performance-driven ones.

The substantive content lives in the three documents above; this README is orientation only.
