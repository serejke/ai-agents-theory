# The rules of this project for comments and documents

`stelint-prose.md` gives the rules for the comments and the Markdown documents
of each project that uses `stelint`. This file gives the rules of this project.
If a rule of this file does not agree with `stelint-prose.md`, obey this file.
`CLAUDE.md` gives the other rules of the book.

The project changes this file. `stelint update` does not change it.

## The reader

The reader is an engineer who uses AI tools, for example Claude Code or ChatGPT.
The reader does not know this theory.

1. Use the words of this reader. Before you use a theory term, tell its meaning
   in usual words and give a known example.
2. Do not make the text more technical than necessary. A short sentence with a
   usual word is better than a long sentence with a special word.
3. Put the most important fact of each section in its first sentence. Then give
   the other facts and the examples.
4. Divide a long section into short sections with clear headings. Put the facts
   that only some readers use at the end of the document.
5. Write each fact one time. If a different document gives the fact, write a
   link to it.

## The structure of a concept document

Each document of `primitives/`, `harness/`, `environment/` and `patterns/` has
these parts in this sequence:

6. The title: `# Name`, with only the name of the concept.
7. The definition: one to three sentences in usual words. A reader who reads
   only this paragraph knows the function of the concept.
8. The composition: one line `**Composition**: ...` in a harness document or a
   pattern document. A document of `environment/` has one line `**Shape**: ...`.
   A document of `primitives/` has no such line.
9. `## Value`: the cause of the concept. Tell how it helps an agent system, and
   the problems that occur without it.
10. `## Definition`: the pseudocode type of the concept.
11. The other sections: variants, examples and decisions.
12. `## Related`: the links to the documents that are directly related.

## Links from other web pages

Persons publish links to the documents of this book on social networks. These
links must continue to operate.

13. Do not move, rename or remove a document.
14. If you change or remove a heading, keep its previous anchor. Put
    `<a id="previous-anchor"></a>` on a line above the new heading, or above the
    text of the previous section. Put an empty line before and after it.
15. The previous anchor is the anchor of GitHub for the previous heading. GitHub
    writes the text in lowercase, removes the punctuation and replaces each
    space with a hyphen.

## Format

16. Run `npx -y prettier@3 --write <file>` after each change. The file
    `.prettierrc.json` sets the maximum line width to 80 columns.
17. stelint does not read code blocks. Keep the pseudocode in TypeScript. Write
    the comments in the code in short and clear English.
18. Write the name of a concept of the theory with an uppercase letter, for
    example "Agent", "Session" and "Tool". Write "agent" and "tool" in lowercase
    for the usual meaning.
19. Write the title of a paper or of a different document in double quotes.
20. Write the names of products, companies and persons as terms of the
    dictionary.
