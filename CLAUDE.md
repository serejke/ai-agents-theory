# AI Agents Theory

A first-principles formalization of the building blocks of AI agent systems. The book answers: what bricks are agent systems assembled from, and what can be built from them?

The canonical taxonomy lives in `README.md`. The filesystem layout at the repo root is the source of truth for what exists; this file does not mirror it.

## Book Quality Rules

These rules define "done" for any document and must hold as an invariant across the entire book. Verify these after every edit.

### Taxonomy Invariants

The theory organizes concepts into four tiers, plus Verification as an orthogonal concern.

- **Primitives** are the concepts that cannot be built from other concepts in the theory. Currently LLM (the stateless reasoning function), Tool (the side-effectful action), and Prompt (the protocol-formatted input the LLM consumes) are the primitives. Do not introduce a new primitive unless it is genuinely irreducible — if you can express a candidate as a composition of existing concepts, it is not a primitive. Prompt has internal slots (system, tools, history, inserts) but is primitive because its structure is defined by the LLM's training protocol, not by other theory concepts; the slots are filled, not composed.
- **Harness** is the running-system skeleton: the machinery that turns primitives into an invocable Agent. Every agent system runs on some instance of this skeleton, whether or not the author thought in these terms. A concept belongs to the harness if a minimal running Agent necessarily has it (iterate the LLM → you have a loop; accumulate turns → you have a session; every tool call flows through a dispatch step where policies could intercept → you have a Guardrail; wrap a session in a frontend and pin it to an environment → you have an Agent). Harness concepts may have registered content that's optional (Guardrail policies, Session history items), but the structural slot itself is not.
- **Environment** is the substrate the agent acts upon — the world the agent inhabits, not part of the agent itself. Tools are the interface; Environment is what the interface reaches. Environment is a distinct ontological tier: it is not a primitive (not irreducible; it's a bundle of resources), not harness (it's not machinery of the agent), and not a pattern (not a compositional choice — every agent has one, even if minimal). Sandboxes, containers, VMs, local OS are variants of Environment.
- **Patterns** are compositions that name a distinct architectural role and are _optional_. A concept earns pattern status when (a) it can be expressed as a composition of primitives, harness, environment, and other patterns, and (b) it names a recurring role that some agent systems adopt and others don't. Memory, StateMachine, Channel, PromptLoading, Workspace, Planner, Router, Evaluator, Continuity, Topology are patterns.
- **Composition formulas are mandatory for harness and pattern documents.** Every harness or pattern document includes a **Composition** line naming what it composes (e.g., "AgentLoop = LLM + Tool[] in a recursive cycle"). Environment documents use a **Shape** line instead — Environment is a resource bundle, not a composition.
- **The canonical taxonomy lives in `README.md` only.** No other document may contain a tier map or taxonomy table. Individual documents link back to the README for the full picture.
- **Classification is stable.** If a concept is classified in a tier, it stays there unless the underlying argument changes. No document may contradict another's classification. If you reclassify something, update every document that references it in the same change.

### Document Format

Every document follows this structure (sections may be omitted if genuinely empty, but the order is fixed):

1. **Title** — `# Name` (just the concept name, no decorators like "deep dive" or "analysis")
2. **Opening definition** — One to three sentences defining what this concept is. Not "the problem" — the definition. A reader who reads only this paragraph knows what the concept is.
3. **Composition** (harness and patterns) or **Shape** (environment) — One line naming what the concept composes, or for environment, what resource bundle it describes. Required immediately after the opening definition. Primitives do not need this line.
4. **Why it matters** — Why this concept exists, what breaks without it, what it enables. Motivation through consequence, not through narrative setup.
5. **Formal definition** — TypeScript-like pseudocode type definition. Every primitive, harness, environment, and pattern must have one.
6. **Body** — Variants, examples, trade-offs, design implications. Structure varies per document.
7. **Related** — Links to related documents. At the bottom, not the top.

### Writing Rules

- **State conclusions, not deliberations.** Never write "is X a primitive or a pattern?" — state which it is and why. The reasoning process belongs in git history, not in the text.
- **No temporal language.** Never write "the most underdeveloped," "doesn't exist yet," "after our deep dive," "updated classification." The book describes the theory as it stands now. Git history preserves evolution.
- **No quantitative framing of the theory.** The theory is qualitative. Do not write "the six primitives," "the four patterns," "three strategies," "five variants" — counts are not what the theory is about, and they date the book as new entries are added. Name items or list them; do not tally them.
- **No "Problem" framing as openers.** Don't open with "The Problem" sections that assume the reader already knows there's a problem. Open with the definition, then explain why it matters.
- **The theory is coherent, not self-challenging.** Never frame new insights as exposing gaps, contradictions, or crises in the theory. New aspects of a concept are part of the concept — they extend and deepen it, not undermine it. Write "Guardrails compose with sandboxes to cover generative tools" not "Guardrails face a crisis with generative tools." The theory describes how things work; it does not argue with itself.
- **Compositional framing over competitive framing.** When two concepts interact, describe how they compose — not which one "wins" or "replaces" the other. Write "Guardrails enforce at the tool boundary; sandboxes enforce at the environment boundary — both layers work together" not "The sandbox becomes the true hard constraint, not the Guardrail."
- **Tier-appropriate phrasing.** When referring to a harness concept, call it "harness" or its own name (AgentLoop, Session, Guardrail, Agent), not "pattern." When referring to a pattern, call it a pattern, not a primitive. Mis-tiering a concept in prose is a classification error.
- **Framework-specific examples illustrate, they don't define.** "Claude Code implements this as X" is a valid illustration. But the definition must be universal — it must hold for any agent system, not just the one used as example.
- **No duplication of canonical content.** The taxonomy map appears once (in README.md). The composition formula for a concept appears once (in that concept's document). If another document needs to reference it, link — don't copy.
- **Integration, not patching.** When adding new insights, rewrite the relevant section to incorporate them. Never append "Note:", "Update:", "Correction:", or "Addendum:" sections. Git history preserves previous versions.
- **All code examples use TypeScript-like pseudocode** (not runnable code). Annotate with comments where the semantics aren't obvious from the types.

### Cross-Reference Rules

- **Relative markdown links only.** `[Memory](../patterns/memory.md)`, not absolute paths or bare names.
- **Every cross-reference must point to an existing document.** Broken links are a build error.
- **Link on first mention per section.** Link a concept the first time it appears in each major section (H2). Don't link every occurrence.
- **No circular dependency in definitions.** Concept A's _definition_ must not require reading concept B whose definition in turn requires A. Definitions are self-contained. Cross-references are for deeper understanding, not for the core definition to make sense.
- **Related section at the bottom** lists documents that provide useful context. Keep it to direct relationships — don't list every document in the book.

### Consistency Checks (verify after every edit)

1. **README matches filesystem.** Every document in the concept folders (`primitives/`, `harness/`, `environment/`, `patterns/`, `verification/`, `case-studies/`) is listed in `README.md` where applicable. Every entry in `README.md` corresponds to an existing file.
2. **Tier matches folder.** Documents under `primitives/` are primitives; documents under `harness/` are harness; documents under `environment/` are the Environment tier; documents under `patterns/` are patterns. Every harness, environment, and pattern document self-identifies with its tier in prose.
3. **Composition or Shape formulas present.** Every harness and pattern document contains an explicit **Composition** line. Every environment document contains a **Shape** line.
4. **No orphan taxonomy claims.** No document contains its own tier map or taxonomy table. The map is in `README.md` only.
5. **No temporal language.** Grep for: "missing," "underdeveloped," "don't exist yet," "will appear," "after deep-dive," "updated classification." None should appear.
6. **No quantitative framing.** Grep for patterns like "the N primitives," "N patterns," "N variants," "N strategies" where N is spelled or numeric. Replace with qualitative phrasing or plain lists.
7. **Tier-appropriate prose.** Harness concepts (AgentLoop, Session, Guardrail, Agent) are referred to as harness, not as patterns. Environment is referred to as its own tier, not as harness or infrastructure. Patterns are referred to as patterns, not as primitives. Grep for mis-tiering when renaming or reclassifying. Use capital-A "Agent" for the formal concept (the runnable unit = Frontend + Session + Environment); lowercase "agent" for colloquial reference to the reasoning (typically the Session level) — same Tool-vs-tool convention applied consistently.
8. **Cross-references resolve.** Every `[text](path.md)` link points to an existing file.
9. **Opening definitions present.** Every document under `primitives/`, `harness/`, `environment/`, and `patterns/` starts with a one-to-three sentence definition immediately after the H1 title.
