# AI Agents Theory

This book gives a model of the parts of AI agent systems. It starts from the
basic facts, not from a product. It tells which parts make an agent system, and
which systems you can build from these parts.

Only `README.md` gives the map of the tiers. The files and folders at the root
of the repository tell which documents the book has. This file does not give
that list again.

## The Language of the Book

The text of the book obeys ASD-STE100 "Simplified Technical English" (STE).

- `.claude/rules/stelint-prose.md` gives the STE rules.
- `.claude/rules/stelint-project.md` gives the rules of this book: the reader,
  the structure of a concept document, the links and the format. If it does not
  agree with this file, obey `.claude/rules/stelint-project.md`.
- `.stelint/dictionaries/project.md` is the dictionary of the book. It is one
  dictionary for all the documents. Use each term only in its meaning in the
  dictionary.

Before each commit, run `stelint` and correct each error.

Persons publish links to the documents of the book on social networks. Thus:

- Do not move or rename a document.
- If you change a heading, keep its previous anchor.
  `.claude/rules/stelint-project.md` tells how, in the section "Links from other
  web pages".

## Book Quality Rules

These rules give the conditions for a correct document. They must stay correct
for all the book. Examine them after each change.

<a id="taxonomy-invariants"></a>

### The Tiers

The theory puts its concepts into these tiers: the primitives, the harness, the
Environment and the patterns. Verification is not a tier. It applies to all the
tiers.

- **Primitives** are the concepts that the theory cannot make from other
  concepts. The primitives are:
  - LLM: the reasoning function that keeps no data between calls
  - Tool: the operation with a side effect
  - Prompt: the input of the LLM in the format of its protocol.

  Add a concept to the primitives only if the theory cannot make it from other
  concepts. A composition of concepts is not one of the primitives. Prompt has
  slots: system prompt, tools, history and inserts. Prompt is one of the
  primitives all the same, because the training protocol of the LLM gives its
  structure, not other concepts. The harness fills the slots. The slots are not
  a composition.

- **Harness** is the structure of a system that runs. It makes an Agent from the
  primitives, and a person or a program can then start the Agent. Each agent
  system runs on such a structure, also if its author does not use these names.
  A concept is a harness concept if each Agent that runs must have it:
  - If you call the LLM again and again, you have a loop.
  - If you keep the turns, you have a Session.
  - Each tool call goes through one point before the tool runs, and policies can
    stop the call there. This point is the Guardrail.
  - If you put a Session in a frontend and connect it to an environment, you
    have an Agent.

  The items in a harness concept can be optional, for example the policies of a
  Guardrail or the items of the history of a Session. The slot for the items is
  not optional.

- **Environment** is the set of resources that the agent changes. It is not a
  part of the agent. The Tools are the interface, and the Environment is the set
  of resources behind the interface. The Environment is a tier, but it is not
  one of the other tiers:
  - It is not one of the primitives, because it is a set of resources that you
    can divide.
  - It is not harness, because it is not a part of the agent.
  - It is not a pattern, because it is not a decision. Each agent has an
    Environment, also if the Environment is very small.

  Sandboxes, containers, virtual machines and the local operating system are
  variants of the Environment.

- **Patterns** are optional compositions that give a name to one function in the
  structure of a system. A concept is a pattern if each of these conditions
  applies:
  - It is a composition of primitives, harness, the Environment and other
    patterns.
  - It gives a name to a function that some agent systems use and other agent
    systems do not use.

  Memory, StateMachine, Channel, PromptLoading, Workspace, Planner, Router,
  Evaluator, Continuity and Topology are patterns.

- **Each harness document and each pattern document has a composition line.**
  The line tells the concepts that it composes, for example "AgentLoop = LLM +
  Tool[] in a recursive cycle". An Environment document has a shape line, not a
  composition line. The cause is that the Environment is a set of resources, not
  a composition.

- **Only `README.md` gives the map of the tiers.** No other document has a map
  of the tiers or a table of the tiers. Each document gives a link to
  `README.md` for the full map.

- **The tier of a concept stays the same.** A concept stays in its tier until
  the argument for that tier changes. Two documents must not give two different
  tiers for one concept. If you move a concept to a different tier, change each
  document that refers to it in the same change.

<a id="document-format"></a>

### The Structure of a Document

Each document has the sections of this list, in this sequence. If a section is
empty, you can remove it. Do not change the sequence.

1. **The title**: `# Name`. Only the name of the concept, with no other words,
   for example "deep dive" or "analysis".
2. **The definition**: one to three sentences that tell the function of the
   concept. Do not give "the problem" here. A reader who reads only this
   paragraph knows the function of the concept.
3. **The composition line** or **the shape line**: one line immediately after
   the definition. A harness document or a pattern document gives the concepts
   that the concept composes. An Environment document gives the resources of its
   set. A primitive has no such line.
4. **`## Value`**: the cause of the concept. Tell the problems that occur
   without it and the functions that it gives. Give the results as the cause. Do
   not start with a long text before the cause.
5. **`## Definition`**: the type of the concept in TypeScript pseudocode. Each
   concept of the primitives, the harness, the Environment and the patterns must
   have one.
6. **The other sections**: variants, examples, decisions and their results for
   the structure of a system. These sections are different in each document.
7. **`## Related`**: the links to related documents, at the bottom of the
   document.

`.claude/rules/stelint-project.md` gives the full format of these sections, in
the section "The structure of a concept document".

<a id="writing-rules"></a>

### The Rules for the Text

- **Give the result, not the process.** Do not write "is X a primitive or a
  pattern?". Give the tier and its cause. The git history keeps the process of
  the decision.
- **Do not write about time.** Do not write "the most underdeveloped", "doesn't
  exist yet", "after our deep dive" or "updated classification". The book gives
  the theory, not the history of the theory. The git history keeps the changes.
- **Do not count the items of the theory.** The theory is about qualities, not
  about numbers. Do not write "the six primitives", "the four patterns", "three
  strategies" or "five variants". A number is not a part of the theory, and it
  becomes incorrect when the book gets new entries. Give the names of the items
  or a list of the items.
- **Do not start with "The Problem".** Such a section thinks that the reader
  knows the problem. Start with the definition, and then tell how the concept
  helps.
- **Help the new reader before you use the words of the theory.** In `README.md`
  and in the first paragraphs of each concept document, tell the meaning of each
  concept in usual words. Use the words of a reader who does not know the
  theory: programs, tasks, tools, filesystems, loops. Then give the name of the
  concept in the theory, as `**Name**`.
  - Do not tell that an Agent is "a Session wrapped in a Frontend and pinned to
    an Environment". A new reader does not know these words.
  - First tell the function of the concept in usual words. Then give a known
    example. Then give the words of the theory.
  - The deep sections of a document can use the words of the theory. The first
    paragraphs must not use them.
  - A reader who opens the document first must know the meaning of the first
    paragraphs without a link.
- **The parts of the theory agree.** Do not show a new fact as a "gap", a
  "contradiction" or a "crisis" of the theory. A new fact about a concept is a
  part of the concept. It makes the concept larger and deeper. Write "Guardrails
  compose with sandboxes to cover generative tools", not "Guardrails face a
  crisis with generative tools". The theory tells how agent systems operate. Its
  parts do not show the theory as incorrect.
- **Tell how concepts compose, not which concept is better.** If two concepts
  operate together, tell how they compose. Do not tell which concept "wins" or
  "replaces" the other concept. Write "Guardrails enforce at the tool boundary;
  sandboxes enforce at the environment boundary — both layers work together". Do
  not write "The sandbox becomes the true hard constraint, not the Guardrail".
- **Give each concept its correct tier in the text.** Call a harness concept
  "harness" or its name (AgentLoop, Session, Guardrail, Agent), not "pattern".
  Call a pattern a pattern, not one of the primitives. An incorrect tier in the
  text is an error of the tiers.
- **An example from a framework shows the concept. It is not the definition.**
  "Claude Code implements this as X" is a correct example. But the definition
  must apply to each agent system, not only to the system of the example.
- **Give each fact in one document only.** Only `README.md` has the map of the
  tiers. Only the document of a concept has its composition line. If a different
  document refers to such a fact, it gives a link and does not give the fact
  again.
- **Write the section again, do not add a patch.** If you add a new fact, write
  the related section again with the fact in it. Do not add sections, for
  example "Note:", "Update:", "Correction:" or "Addendum:". The git history
  keeps the previous versions.
- **Write all code examples in TypeScript pseudocode.** The code does not run.
  If the types do not show the meaning, add comments.

<a id="cross-reference-rules"></a>

### The Rules for Links

- **Use only links with a path from the folder of the document.** Write
  `[Memory](../patterns/memory.md)`. Do not write an absolute path or a name
  without a link.
- **Each link must go to a document of the book.** A broken link is a build
  error.
- **Give a link to a concept the first time that it occurs in each `##`
  section.** Do not give a link each time.
- **Do not make circular definitions.** If the definition of a concept uses a
  second concept, the definition of the second concept must not use the first
  concept. Each definition must be clear without other documents. The links give
  more knowledge, but the definition must be clear without them.
- **Put the `## Related` section at the bottom.** Give the documents that help
  the reader. Give only the documents that are directly related, not all the
  documents of the book.

<a id="consistency-checks-verify-after-every-edit"></a>

### The Checks After Each Change

1. **`README.md` agrees with the filesystem.** Where applicable, `README.md`
   gives each document of the concept folders: `primitives/`, `harness/`,
   `environment/`, `patterns/`, `verification/` and `case-studies/`. Each entry
   of `README.md` has a file.
2. **The tier agrees with the folder.** A document of `primitives/` gives one of
   the primitives. A document of `harness/` gives a harness concept. A document
   of `environment/` gives the Environment tier. A document of `patterns/` gives
   a pattern. Each harness document, Environment document and pattern document
   tells its tier in the text.
3. **The composition lines and the shape lines are in their documents.** Each
   harness document and each pattern document has a `**Composition**` line. Each
   Environment document has a `**Shape**` line.
4. **No other document has a map of the tiers.** Only `README.md` has the map.
5. **No document has words about time.** Use `grep` to find these words:
   "missing," "underdeveloped," "don't exist yet," "will appear," "after
   deep-dive," "updated classification." The book must not contain them.
6. **No document counts the items of the theory.** Find a number before
   "primitives," "patterns," "variants" or "strategies" with `grep`. The number
   can be in digits or in words. Write the text again without the number, or
   with a list.
7. **The text gives each concept its correct tier.** The harness concepts
   AgentLoop, Session, Guardrail and Agent are harness, not patterns. The text
   calls the Environment a tier, not harness and not "infrastructure". The
   patterns are patterns, not primitives. After you rename a concept or move it
   to a different tier, find incorrect tiers with `grep`.
8. **Each link goes to a file.** Each `[text](path.md)` link goes to a file of
   the book.
9. **Each concept document starts with its definition.** Each document of
   `primitives/`, `harness/`, `environment/` and `patterns/` has a definition of
   one to three sentences immediately after the `#` title.

Write "Agent" with an uppercase letter for the concept of the theory. It is the
program that runs, made of a frontend, a Session and an Environment. Write
"agent" in lowercase for the usual meaning, usually the reasoning at the level
of the Session. "Tool" and "tool" follow the same rule.
