# Case Study: MCP Registries and the Automation Pivot

Two movements in the agent capability space are heading toward the same destination from opposite directions: MCP registries (bottom-up, developer-driven) and automation platform pivots (top-down, business-driven). Understanding where each is strong and weak reveals what the actual capability marketplace will look like.

---

## MCP Registries

### What They Are

MCP (Model Context Protocol) server registries are directories of [Tools](../primitives/tool.md) that agents can discover and call. Think npm, but for agent capabilities instead of JavaScript packages. A developer publishes an MCP server (e.g., "query EU consumer protection law"), registers it with metadata (description, input/output schemas, pricing), and any compatible agent can find and use it.

### Current State

- **Several registries exist** — mcp.so, Smithery, Glama, and others index hundreds of community MCP servers. Anthropic maintains a reference list. No single dominant registry yet.
- **Entirely developer-oriented** — installing an MCP server means editing JSON config files, running local processes, managing dependencies. No non-developer has ever done this.
- **No standardized quality signals** — no ratings, no reliability metrics, no usage stats. You find a server, read its README, hope it works. This is npm circa 2012.
- **No payment layer** — every MCP server today is free. No mechanism for paid capabilities, usage metering, or billing. The protocol itself has no concept of pricing.
- **Local-first architecture** — most MCP servers run on the user's machine, connecting to remote APIs. This limits what's possible (no server-side state, no background processing) but maximizes privacy and control.

### What's Missing

The gap between "developer tool registry" and "capability marketplace" is enormous:

| Have                     | Need                                                                                           |
| ------------------------ | ---------------------------------------------------------------------------------------------- |
| JSON schema descriptions | Semantic capability discovery (agent understands what the tool _does_, not just its API shape) |
| README documentation     | Machine-readable quality metrics (uptime, latency p99, accuracy scores)                        |
| GitHub stars             | Usage-based reputation — how often agents choose this tool, success rate of invocations        |
| Free/open source         | Pricing metadata, usage metering, billing integration                                          |
| Manual installation      | One-click or agent-initiated provisioning                                                      |
| Local execution          | Hosted execution with SLAs, or hybrid local/remote                                             |

### The npm Analogy — and Its Limits

MCP registries look like early npm, and that's both promising and concerning.

**Promising:** npm proved that a community-driven, open registry can scale to millions of packages and become critical infrastructure. The MCP ecosystem could follow the same path — permissionless publishing, organic quality signals, composability through shared protocols.

**Concerning:** npm also proved that open registries have serious problems — supply chain attacks, dependency hell, abandoned packages, no quality guarantees. For agent capabilities that handle user data and real-world actions (filing complaints, sending emails, making payments), the stakes are much higher than a broken build. A malicious MCP server could exfiltrate data, impersonate the user, or take destructive actions.

**The likely resolution:** a tiered model. Open registries for development and experimentation. Curated/certified registries for production use with user data. Platform-managed registries (Anthropic's own, OpenAI's own) for the highest-trust tier. Similar to how npm has the public registry, private registries, and enterprise solutions.

---

## The Automation Platform Pivot

### Zapier, Make, n8n — What They Have

These platforms spent years building something incredibly valuable: **thousands of pre-built, maintained, authenticated integrations with real services.**

- **Zapier:** 7,000+ app integrations. OAuth flows, webhook handlers, data transformations, error handling — all production-hardened.
- **Make (formerly Integromat):** 1,500+ integrations with visual workflow builder. Strong in complex multi-step scenarios.
- **n8n:** Open-source alternative, 400+ integrations, self-hostable. Developer-friendly, code-extensible.

This integration library is their moat. Building and maintaining OAuth flows, handling API versioning, managing rate limits, dealing with provider-specific quirks — this is unglamorous but essential work that takes years to accumulate.

### The Pivot

All three are repositioning from "automation platform you build workflows in" to "agent tool provider":

- **Zapier** launched "AI Actions" — exposing their integration library as callable tools for AI agents. An agent can say "send an email via Gmail" and Zapier handles the OAuth, the API call, the error handling. The agent never touches Gmail's API directly.
- **Make** is building MCP server support, letting agents trigger Make scenarios as tools.
- **n8n** added an "AI Agent" node and tool-calling capabilities, positioning workflows as composable agent tools.

### Why This Is Strategically Smart

The automation platforms realized something crucial: **their UI is the part that gets inverted, but their integration layer is the part that persists.** This is a textbook [App Inversion](architecture.md) play.

```
Traditional Zapier:  User → Zapier UI → Trigger → Action → Service
Agent Zapier:        User → Agent → Zapier Tool → Action → Service
```

The "Zapier UI" layer (visual workflow builder, trigger configuration, testing interface) is exactly what agents replace. But the "Action → Service" layer (authenticated API calls to 7,000 services) is exactly what agents need and can't easily replicate.

### Why This Is Also Risky

1. **Margin compression.** Zapier charges $20-600/month for workflow automation. If their value reduces to "make API calls on the agent's behalf," they're competing with raw API access. The visual builder and workflow logic were the premium. Remove those, and what justifies the price?

2. **Disintermediation.** Once agents can call APIs directly (via MCP servers or native tool use), why route through Zapier? Zapier's value is "you don't need to understand Gmail's API." But agents understand APIs natively — they can read docs, generate auth flows, handle errors. The abstraction layer becomes redundant for capable agents.

3. **Competition from MCP native.** Every service will eventually publish its own MCP server. Google will publish a Gmail MCP server. Slack already has one. When the source provides the tool directly, the middleware loses its reason to exist.

4. **Lock-in reversal.** Zapier's lock-in comes from workflows — complex automations that are painful to rebuild elsewhere. Capabilities don't have this lock-in. If an agent can call Zapier's Gmail tool or Google's native Gmail MCP server, switching cost is zero.

### The Survival Path

Zapier-like platforms survive if they become the **reliability and trust layer** rather than the integration layer:

- **Guaranteed uptime and SLAs** — "call Gmail through us, and we guarantee 99.9% reliability with automatic retry and fallback." Raw API access doesn't offer this.
- **Compliance and audit** — "every action your agent takes through us is logged, auditable, and reversible." Enterprise requirement that raw tools can't satisfy.
- **Complex multi-step orchestration** — some workflows are genuinely complex (conditional branching, data transformation, error recovery). Packaging these as single composite tools is more valuable than exposing raw API calls.
- **Pre-authenticated access** — managing OAuth tokens, refresh flows, and credential rotation across thousands of services is a real operational burden. If Zapier handles this, it saves every agent platform from reimplementing it.

---

## Where They Converge

MCP registries and automation platforms are approaching the same destination from opposite directions:

|                | MCP Registries                                    | Automation Platforms                                       |
| -------------- | ------------------------------------------------- | ---------------------------------------------------------- |
| **Origin**     | Protocol/developer community                      | Business/product companies                                 |
| **Strength**   | Open, composable, protocol-native                 | Battle-tested integrations, auth, reliability              |
| **Weakness**   | No quality guarantees, no payment, no trust layer | UI-centric legacy, margin pressure, disintermediation risk |
| **Discovery**  | Registry search, manual config                    | Platform-mediated, curated                                 |
| **Auth model** | User manages own credentials                      | Platform manages OAuth on user's behalf                    |
| **Pricing**    | Free (today), usage-based (future)                | Subscription (today), per-invocation (future)              |

### The Most Likely Outcome

Neither wins alone. The capability marketplace that emerges will have **layers**:

1. **Protocol layer (MCP)** — the universal interface. Every capability speaks MCP (or whatever protocol wins). Commodity infrastructure, like HTTP.
2. **Registry layer** — discovery and metadata. Multiple registries coexist, some open (npm-like), some curated (App Store-like), some platform-specific.
3. **Trust layer** — quality signals, reliability metrics, security certification. This is where Zapier-like companies can reposition.
4. **Payment layer** — usage metering, billing aggregation, revenue sharing. Likely owned by agent platforms (Anthropic, OpenAI) initially, then potentially decentralized.
5. **Capability layer** — the actual services. Mix of direct-from-source (Google's own Gmail tool), middleware (Zapier-wrapped integrations), and independent providers (specialized domain services).

### The Stripe Analogy

The best historical parallel might be Stripe, not the App Store.

Stripe didn't build a marketplace — it built **infrastructure that made payments invisible.** Before Stripe, accepting payments required merchant accounts, PCI compliance, and complex integrations. Stripe reduced it to seven lines of code.

The winning play in agent capabilities might be similar: not building a marketplace where users browse, but building infrastructure that makes capability discovery, invocation, and payment **invisible to both the user and the agent.** The agent just says "file a complaint about this flight" and the infrastructure handles discovery, selection, authentication, invocation, payment, and quality monitoring.

---

## Open Questions

- **Will MCP remain the dominant protocol, or will competing standards fragment the ecosystem?** OpenAI hasn't fully committed to MCP. Google has their own approach. Protocol wars could slow everything down.
- **Can automation platforms pivot fast enough?** Their revenue models, organizational structures, and product cultures are built around visual builders and human users. Pivoting to agent-first is a fundamental transformation, not a feature addition.
- **Who owns the trust layer?** This might be the most valuable piece — the entity that certifies capabilities as safe, reliable, and compliant. Could be agent platforms, could be independent (like certificate authorities for HTTPS), could be regulatory.
- **When do capabilities start competing on price?** Today everything is free or bundled into subscriptions. The moment capabilities become individually priced and agent-discoverable, price competition begins and margins collapse. This is the moment the market truly forms.

---

## Related

- [Architecture](architecture.md) — why the integration layer persists while the UI layer inverts
- [Economics](economics.md) — the market dynamics driving this convergence
- [GPT Store](case-gpt-store.md) — the first failed attempt at a capability marketplace
