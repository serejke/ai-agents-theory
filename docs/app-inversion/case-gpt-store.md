# Case Study: GPT Store — Apps in Agent Clothing

The GPT Store is a case study in applying the wrong model to agent capabilities. It failed not because the timing was wrong, but because it replicated the app model inside an agent interface — exactly what [App Inversion](architecture.md) predicts will fail.

---

## What Happened

OpenAI launched the GPT Store in January 2024 as "build custom GPTs and share them." The pitch: anyone can create a specialized AI assistant, publish it, and eventually monetize it. The App Store for AI.

It didn't work.

---

## Five Structural Problems

### 1. No Composition

You couldn't tell ChatGPT "use the Resume Writer AND the Job Search GPT together to tailor my application." Each GPT was a silo.

In a true capability model, these would be [Tools](../primitives/tool.md) that one agent orchestrates together. The user says "help me apply for this job" and the agent calls a resume-formatting capability, a job-requirements-analysis capability, and a cover-letter capability — all in one conversation, composing their outputs.

The GPT Store forced the user to be the orchestrator — manually switching between GPTs, copy-pasting context. This is worse than the app model, because at least apps can share data via APIs.

### 2. Discovery Was App-Store-Shaped

Users browsed categories, read descriptions, picked one. But agents should discover and select capabilities **programmatically based on user intent**, not require the user to shop.

The discovery model assumed humans would evaluate which GPT to use — defeating the purpose of having an agent in the first place. In a capability marketplace, the agent evaluates options based on reliability metrics, pricing, and task fit. The human never sees the "store."

### 3. The Value Was in the Prompt, Not the Capability

Most GPTs were just system prompts with some uploaded PDFs. No proprietary data, no API integrations, no real service behind them. A true capability provides something the agent **can't do on its own** — access to a database, a licensed API, domain-specific computation, real-world actions.

A "Legal Advisor GPT" that's just a prompt saying "you are a legal expert" adds nothing over asking ChatGPT directly with good instructions. A legal capability that queries a jurisdiction-specific statute database, calculates filing deadlines, and generates properly formatted court documents — that's a real capability with a defensible moat.

### 4. No Interoperability

GPTs couldn't call each other, share context, or compose into workflows.

Compare this to MCP tools, where an agent can call a flight-lookup tool, then a complaint-filing tool, then an email-drafting tool — all in one conversation, with context flowing between them via [Channels](../primitives/channel.md). The protocol enables composition; the GPT Store actively prevented it.

### 5. Wrong Economic Unit

The GPT Store tried to monetize whole "apps" (subscriptions, revenue sharing per GPT). But the natural economic unit in an agent world is the **capability invocation**, not the **agent instance**.

Nobody pays $5/month for "Resume Writer GPT" when they write a resume twice a year. But they'd happily pay $0.50 per resume formatted, or $2 per job application optimized. Usage-based pricing aligns with how agents consume capabilities — sporadically, in bursts, composed with other capabilities.

---

## The Result

The store filled with thousands of thin wrappers barely distinguishable from a good prompt. Users had no reason to leave the main ChatGPT interface for a slightly customized version of it. Creators had no way to build real defensibility. Revenue sharing never materialized meaningfully.

---

## The Lesson

The GPT Store failed because it answered the wrong question. It asked: _"How do we let people build and sell custom AI assistants?"_ The right question is: _"How do we let agents discover and compose external capabilities on behalf of users?"_

| GPT Store (wrong model)     | Capability marketplace (right model)               |
| --------------------------- | -------------------------------------------------- |
| User browses and picks      | Agent discovers and selects                        |
| Each GPT is a silo          | Capabilities compose freely                        |
| Value = prompt + persona    | Value = proprietary data + real backend            |
| Monetize the agent instance | Monetize the capability invocation                 |
| Human evaluates quality     | Agent evaluates metrics (latency, accuracy, price) |

The GPT Store was Web 1.0 portal thinking applied to the agent era. What's needed is the equivalent of the open web — composable, discoverable, interoperable services that agents navigate on the user's behalf.

---

## Related

- [Architecture](architecture.md) — the structural analysis that predicts why the GPT Store model fails
- [Economics](economics.md) — the economic forces driving capability marketplaces
- [MCP Registries](case-mcp-registries.md) — what's emerging instead
