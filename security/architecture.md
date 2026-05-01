# Architecture

This document is the operational counterpart to [First Principles](first-principles.md). It maps the conceptual axes — process authority, agent authority, the lethal trifecta — to concrete architectural choices: sandbox tiers and what each buys you, the defense families that have actual evidence, what major vendors ship in production versus what they research, and a working checklist for an agent design.

## Sandbox Tiers

Sandboxes are the operational answer to process authority. They are an [Environment](../environment/environment.md)-level constraint, not a [Guardrail](../harness/guardrail.md). The ladder, ordered by isolation strength against cold-start cost, with what each tier actually delivers:

| Tier                                                           | Cold start | What it solves                                                                                              | What it does not                                                                               |
| -------------------------------------------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Process-level (seccomp, AppArmor, Seatbelt)                    | Negligible | Specific syscall classes (`ptrace`, raw sockets, mount namespaces)                                          | Anything the agent does via `openat()` or HTTP — i.e., nearly everything that matters          |
| Container (Docker, Podman, containerd)                         | 100ms–2s   | Filesystem and process-tree separation, _if_ configured without host mounts, host network, or Docker socket | Kernel-level escape (shared kernel; CVE-2024-21626 "leaky vessels" was the canonical reminder) |
| gVisor                                                         | ~1s        | The above plus user-space syscall filter                                                                    | High syscall latency (2–10×); shares some kernel surface                                       |
| microVM (Firecracker, Cloud Hypervisor, Kata Containers)       | ~125ms     | The above plus separate guest kernel — kernel escape isolated                                               | Authority leakage; prompt injection; everything in the agent-authority axis                    |
| Language sandbox (Pyodide/WASM, V8 isolates, Deno permissions) | 5–50ms     | Filesystem and network simply do not exist in the runtime by default                                        | Limited library ecosystem; many agent workloads need native tools                              |
| Ephemeral cloud sandbox (e2b, Modal, Daytona, Vercel Sandbox)  | 200ms–2s   | Productized microVM-per-task with hibernation; pay-per-use; per-task isolation by default                   | Same as microVM — nothing on agent authority                                                   |

Two consensus moves became default in 2026:

- **Per-task microVM** is the standard for any multi-tenant agent execution. Drivers: ChatGPT Code Interpreter sandbox-escape demos (2023) pushed shared-container vendors to per-session VMs; CVE-2024-21626 made microVM-by-default the norm where tenants are not trusted; the productization of e2b, Vercel Sandbox, and Cloudflare Sandbox SDK in 2025 made adoption cheap. Resource overhead (~50–200MB RAM per VM, ~125ms boot) is acceptable for agent workloads.
- **Per-MCP-server sandbox** is table-stakes after multiple supply-chain incidents in 2025 and early 2026. The MCP protocol itself does not provide capability scoping; the operator does it at the host level by running each server in its own isolated environment.

What microVMs do not do is know about the agent's authority. The Replit production-database deletion of July 2025 happened inside a sandbox that perfectly contained the agent's process. The agent had the database credential; the sandbox proxied the destructive query without complaint. Process isolation answered "can the agent's process touch my disk?" — the actual question was "should the agent be able to issue this query with these credentials?" The latter is a [Guardrail](../harness/guardrail.md) question, not an Environment question. See [References § Replit production DB deletion](references.md#replit-production-db-deletion).

## The Defense Families That Have Evidence

Defenses split cleanly into two camps. Probabilistic defenses live inside the model — adversarial training, instruction-hierarchy fine-tuning, classifier sidecars, output filters. They reduce per-attempt success rate; they do not change the asymptote, and they degrade against adaptive attackers. Architectural defenses live around the model and treat the model as untrusted; they substitute deterministic mechanisms for the model's compliance.

The serious 2025 papers all share an assumption: **treat the LLM as untrusted and put deterministic guard rails outside it.** This is the camp with provable claims and the camp the field is converging on.

### Capability-Tagged Plan Execution (CaMeL Family)

CaMeL — Debenedetti, Tramèr, Carlini, Shumailov et al. at DeepMind, March 2025 — is the most rigorous published defense and the cleanest exemplar of the architectural-defense pattern. The mechanics:

1. A privileged Planner LLM sees only the trusted user query. It emits a program in a restricted Python-like DSL whose only operations are calls to whitelisted tools and dataflow between values.
2. A non-LLM Custom Interpreter executes that program. Untrusted data flows through the interpreter as opaque values.
3. A Quarantined LLM is invoked when natural-language transformation is needed (extract an address from an email body, translate a snippet) — but it has no tool access; its output is bound back as values, not as program edits.
4. Each value carries capability metadata: a `readers` set (confidentiality) and a `sources` set (integrity). Tool calls are authorized by deterministic policy checks against those tags.

The guarantee: any attack class whose goal is to change control flow or insert new tool calls by injecting instructions through retrieved data is mitigated by construction. The plan is fixed before untrusted data is read; injected instructions cannot be promoted to actions. Crucially, the guarantee holds _even if both LLMs are fully compromised by injection_, because the privileged Planner never sees attacker bytes, and the Quarantined LLM has no tool authority.

The cost: about seven percentage points of utility on the AgentDojo benchmark (77% task completion under CaMeL versus 84% undefended). Two model calls per data-touching step plus interpretation overhead. Not free; not unreasonable for a class of guarantees that no probabilistic defense provides.

What CaMeL does not solve: data-flow exfiltration through arguments the planner already authorized — the planner says "send email to extracted_address" and the Quarantined LLM is tricked into returning the attacker's address. Capability labels on integrity sources are supposed to refuse this; whether they do correctly depends on policy authorship. Capability systems move the problem from "hope the model resists" to "specify the policy correctly," which is a much better problem to have.

See [References § CaMeL](references.md#camel).

### Information-Flow Control (Fides Family)

Fides — Costa, Köpf et al. at Microsoft Research, May 2025 — runs the same pattern with a stronger formal vocabulary. A planner tracks confidentiality-and-integrity labels on every value, deterministically enforces a security policy, and adds primitives to _selectively hide_ information from sub-planners. The paper is explicit about what dynamic taint-tracking _cannot_ catch (implicit flows, semantic leaks), which is itself useful — calibration of the guarantee.

A companion line, Permissive Information-Flow Analysis (Siddiqui et al., October 2024), tackles the classical pathology of taint tracking: over-tainting. Naive IFC says "any output touched by tainted input is tainted," which collapses every retrieved-content workflow to "untrusted." The permissive variant propagates only labels of inputs that _influenced_ the output, measured via attention or k-NN influence. Reported as improvement over baseline in over eighty-five percent of cases.

The negative-result counterweight: Wang et al.'s NeuroTaint paper (2026) shows that classical Bell-LaPadula symbolic IFC achieves F1 ≈ 0.522 on their TaintBench, compared with 0.928 for an approach modeling semantic transformation, causal influence, and cross-session memory persistence. The takeaway for an architect: classical labels are necessary but undersell the propagation graph in an LLM context; expect the field to move toward semantically-aware tracking.

See [References § Fides](references.md#fides).

### Plan-Then-Execute and the Design-Patterns Catalog

Beurer-Kellner, Tramèr, Debenedetti et al. published "Design Patterns for Securing LLM Agents Against Prompt Injections" in June 2025. It is the closest thing the field has to a textbook. It catalogs design patterns each of which, by construction, eliminates a class of injection attacks: Action-Selector, Plan-Then-Execute, LLM Map-Reduce, Dual LLM, Code-Then-Execute (CaMeL is an instance), and Context-Minimization. Each pattern is presented with the threat it removes and the workloads it does and does not fit.

The unifying claim: commit to a tool-call plan _before_ reading any untrusted content; execute the plan with a non-LLM dispatcher; data only re-enters as values, never as new instructions. CaMeL is the strongest instance; weaker instances (a planner LLM emits a structured tool-call plan; a non-LLM dispatcher executes it; outputs return as values) are implementable today without the full capability machinery.

What plan-first does not solve: argument-level data flow leaks. A plan `for each email do summarize(email); maybe_send(reply)` still has an injection in one email corrupt the reply. Plan-first is necessary but not sufficient; combine with capability-tagged argument binding to cover the data-flow leg.

See [References § Design Patterns paper](references.md#design-patterns-paper).

### Dual-LLM (Origin Pattern)

Simon Willison's Dual-LLM pattern (April 2023) is the spiritual ancestor of CaMeL. A privileged LLM never sees raw untrusted bytes; a quarantined LLM sees them but has no tools and emits opaque symbolic placeholders (`$VAR1`, `$VAR2`); binding from symbol to real value happens deterministically outside both models.

Pure Dual-LLM does not survive contact with tasks that require the planner to _reason about_ untrusted content (e.g., "if the email is angry, escalate"). CaMeL is the productionized refinement: the planner emits code rather than English, and the symbolic algebra becomes a typed dataflow program operated on by a deterministic interpreter.

Status: nobody ships pure Dual-LLM in production. The pattern survives in spirit inside CaMeL, AgentDojo's "tool filter" defense, and OpenAI's source-sink framing.

### Capability and Object-Capability Frameworks for Agents

Object-capability security applied to agents is the natural next move. MiniScope (December 2025) frames tool authorization as a capability problem: each agent task receives a sub-capability set carved mechanically from the parent's, not via prompt instruction. AgentGuardian (2026) attempts to _learn_ access-control policies from observed behavior — which is interesting but smells: learning a security policy from a possibly-already-compromised agent is suspect on its face.

The "permission-as-a-tool" pattern — Claude Code's permission prompts, Cursor's approval gates — exists in production today in toy form. Nobody has shipped a tool-call API where the _call itself carries an unforgeable capability handle attenuated by the parent agent_. That is the open frontier; MiniScope is pointing at it.

This is, in the theory's vocabulary, [Topology](../patterns/topology.md) plus [Guardrail](../harness/guardrail.md): the capability set is a structural Topology of which subagent or tool-instance gets which authorities; the enforcement is a Guardrail policy at the dispatch boundary that refuses calls outside the held capabilities.

## What Vendors Ship in Production

There is a gap between research and production. Every major vendor has a defense stack; none of them ship a CaMeL- or Fides-class provable architecture. The shipped defenses raise attacker cost; they do not change the asymptote.

| Vendor                    | Stack                                                                                                                                                                                                                                                                                                                |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Anthropic / Claude Code   | Constitutional refusal training; tool-permission gates with ask-once / always-allow UX; command blocklist; read-only-by-default file access; per-session isolated VMs in cloud builds; a server-side prompt-injection probe scans tool outputs before they enter the agent's context (Auto Mode).                    |
| OpenAI / Atlas / Codex    | Instruction-hierarchy training (Wallace et al., 2024) treats system > user > multimodal > tool output; Watch Mode pauses agents on sensitive sites if user attention drifts; an RL-trained social-engineering red-team model; source–sink combinatorial analysis as user-facing guidance. Public stance: not solved. |
| Google / Gemini / Mariner | ML classifiers on inputs; surrounding "stay focused" instruction tags; RLHF on injection corpora; Workspace-side controls. DeepMind's published lessons paper (May 2025) is candid that adaptive attackers regain ground.                                                                                            |
| Microsoft Copilot         | XPIA classifier on cross-prompt-injection attempts (default-on, not user-disable); Azure Prompt Shields at the AI Content Safety layer; Defender hooks. EchoLeak (CVE-2025-32711) defeated this stack in production.                                                                                                 |
| MCP protocol              | OAuth 2.1 framework; transport guidance; "user consent for tool invocation" as a recommendation. The protocol punts almost everything to the client. Tool-poisoning and rug-pulls are documented unfixed-at-protocol-level.                                                                                          |

Two operational read-throughs from this table:

**The shipped stacks are heuristic-stacking defense in depth.** No vendor has shipped a structural guarantee. Every layer in every shipped stack is independently bypassable; the bet is that an attacker has to bypass several. The bet is reasonable for raising cost but does not eliminate the threat.

**EchoLeak is the proof-of-fail.** Microsoft's stack is the most aggressive of the four — a dedicated XPIA classifier, link redaction, content-safety layer — and a single crafted email zero-clicked it through reference-style markdown, auto-fetched images, and a Teams CSP-allowed proxy. Classifier-based defense at the production scale of M365 was demonstrably bypassed. Treat any "we use a classifier" claim as risk reduction, not as a structural defense.

See [References § Anthropic prompt-injection defenses](references.md#anthropic-claude-code-defense-stack), [§ OpenAI hardening Atlas](references.md#openai-hardening-atlas), [§ DeepMind Gemini lessons](references.md#deepmind-gemini-lessons-blog), [§ Microsoft XPIA](references.md#microsoft-xpia-defense).

## Negative Results Worth Knowing

Knowing what does _not_ work is as important as knowing what does. Several defenses are popular, plausible-sounding, and demonstrably bypassed.

- **Spotlighting** (Hines et al., Microsoft, March 2024) — datamarking, encoding, delimiters to mark untrusted content. Drops attack success rate below two percent on stock prompts, then is bypassed by adaptive attacks. Useful as one heuristic layer; unsafe as the structural defense.
- **Fine-tuning to refuse injection does not generalize.** Zhan et al.'s NAACL Findings 2025 paper evaluated eight defenses and broke all eight with adaptive attacks; success rate stayed above fifty percent on every one. Pasquini's "The Attacker Moves Second" (October 2025) makes the point sharper still: any fixed defense gets bypassed by an attacker who knows the defense.
- **Constrained decoding can be weaponized.** DictAttack (Zhang et al., March 2025) hides the malicious payload in the _grammar_ (control plane) while keeping the prompt benign (data plane), achieving 94–99.5% success on GPT-5, Gemini 2.5 Pro, DeepSeek-R1, and GPT-OSS-120B. Constraining tool-call output to JSON does not constrain semantics.
- **Guardrail SLMs are evadable by Unicode tricks.** Homoglyphs, zero-width characters, and character smuggling defeat classifier-style detectors; this is well-documented through 2025.
- **EchoLeak defeated Microsoft's _dedicated_ XPIA classifier.** Real-world bypass at production scale. The architectural lesson: classifier-based defenses are bypassable, full stop.

See [References § Negative-result papers](references.md#negative-results).

## A Working Checklist for an Agent Design

This is a stance for a 2026 agent architect designed to survive scrutiny. It is not a complete cookbook; it is a set of decision points that, when each is answered well, produce an agent whose security posture is defensible.

**Apply the lethal-trifecta filter at design time.** For every code path through the agent, identify which of the three legs (private data, untrusted content, external comms) it touches. If a single code path touches all three, it is exfiltratable; sever a leg by construction. Examples: an email-summarization agent that only reads and never writes (no external comms); a coding agent on a public repo with no proprietary data (no private data); a research agent that browses but operates against a sanitized data-only context (no untrusted content reaches the privileged planner).

**Sandbox is necessary, not sufficient.** Run code in per-task microVMs as the default. Run each MCP server in its own sandbox. Treat the sandbox as bounding process authority only — it does nothing about agent authority. The Replit and EchoLeak incidents both happened inside sandboxes.

**Where data-flow integrity matters, use capability-tagged execution.** CaMeL is the published frontier; nobody ships it yet. A weaker but implementable version today: a privileged planner LLM emits a structured tool-call plan; a non-LLM dispatcher executes it; outputs return as values, never as instructions. This rules out the largest class of indirect injection — control-flow hijack via retrieved content.

**Treat MCP servers, skills, plugins as a hostile supply chain.** Pin and audit the way you would audit native dependencies, plus content-scan for natural-language payloads. Sandbox each server at the host level. Do not run untrusted skills with the agent's default authority; give them a least-privilege subset issued per task.

**Layer probabilistic defenses but assume the LLM and the classifier both lie.** Adversarial training, prompt-injection probes, instruction-hierarchy hardening, output filters all buy useful risk reduction. None of them are the structural defense. Compose them with the structural defenses above; do not rely on any of them alone.

**Calibrate per-attempt rates against scale.** A defense that takes per-attempt success from fifty percent to one percent is real progress and yet still not enough at scale. For an agent processing thousands of untrusted documents per day, the relevant unit is compounded probability over the day, not per-attempt rate. Plan for the long-tail compromise; do not plan around the modal turn.

**Make output a first-class attack surface.** The agent's reply is a payload the user trusts. Markdown image rendering, auto-fetched URLs, hyperlink display text, code blocks that look benign — all are attack vectors when the output is generated from poisoned context. The user-facing renderer is part of the trust boundary; treat it that way. Strip auto-fetched URLs by default. Render only after a Guardrail post-check on the output.

**Bound blast radius via Topology, not via the model's restraint.** Use [Topology](../patterns/topology.md) — subagent splits, capability attenuation parent-to-child, separation of read-and-summarize agents from take-action agents — to ensure that if one agent is compromised, its authority is small. Do not give a single agent the trifecta and rely on its training to refuse.

The honest 2026 architecture: capability-tagged dataflow plus plan-first execution plus classifier as belt plus sandbox as suspenders. Model-layer and classifier-layer defenses are belt-and-suspenders for agent authority; they cannot be the structural defense. The structural defense is removing the trifecta by construction, and there is currently no other known structural defense.

## Related

- [First Principles](first-principles.md) — the conceptual frame for the choices above.
- [References](references.md) — papers, vendor publications, and incidents cited above.
- [Environment](../environment/environment.md) — sandbox tiers as Environment variants.
- [Guardrail](../harness/guardrail.md) — the harness layer at which capability-policy enforcement composes with structural defenses.
- [Topology](../patterns/topology.md) — security-driven Topology choices (planner/executor splits, dual-LLM, capability-attenuated subagents).
