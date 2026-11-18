<!--
stelint 0.3.0 wrote this file. Do not change it. `stelint update` writes
it again from the text of the installed stelint. Write the rules of this
project in stelint-project.md.
-->

# The prose that stelint checks

This file tells how to write the prose that stelint checks: the comments
and the Markdown documents of the files that `.stelint/config.toml`
selects. Write them in ASD-STE100 Simplified Technical English (STE) with
the project dictionary.

The reader is a person who knows the language of the code and does not
know this project. The reader can also be an agent that reads each word in
its dictionary meaning. Each word has one meaning, and each sentence gives
one fact or one instruction.

`stelint-project.md` gives the rules of this project. If a rule of
`stelint-project.md` does not agree with a rule of this file, obey
`stelint-project.md`.

## stelint and its configuration

1. `.stelint/config.toml` gives the files of the check, the word lists and
   the limits of the rules. If it gives a different limit, obey it.
2. Each Markdown file of `.stelint/dictionaries/` is a dictionary. A
   dictionary without paths applies to each file. A dictionary with paths
   applies only to the files that match its paths, and its first paragraph
   gives these paths. stelint also reads each word list of
   `.stelint/words/`.
   - A dictionary in your context does not apply to each file. Use a term
     only in the files of its dictionary.
   - `stelint dictionary resolve <path>` gives the dictionaries of a file.
3. A rule id in parentheses, for example (`sentence-length`), identifies
   the rule of stelint that checks the instruction. stelint cannot check an
   instruction without a rule id. Obey it too.
4. Do not put a word in backticks or in double quotes only because stelint
   reports it. stelint does not read the words in backticks or in double
   quotes.

## Words

5. Use only these words (`vocabulary`):
   - the approved words of the word lists of `.stelint/config.toml` and
     `.stelint/words/`
   - the technical nouns and technical verbs of the dictionaries of the
     file.
6. Use each word only in its approved meaning. stelint cannot read the
   meaning of a word.
7. Put each term in the dictionary with the smallest scope. That scope
   holds each file that uses the term for the same object. Give the meaning
   of each term one time.
   - A dictionary with paths is for one module, or for a group of modules
     with special words. Put each other term of two or more modules in the
     project dictionary.
   - Do not add a term to a second dictionary. If a second scope uses the
     term for that object, move it to a dictionary that holds the two
     scopes.
   - If two scopes use one word for two objects, give the word a row in the
     dictionary of each scope. Do not put it in a dictionary that holds the
     two scopes. If all the project uses one of the objects, keep its term
     in the project dictionary. Give the other scope a longer term.
   - A file of the two scopes uses a term of a larger scope or a longer
     term, for example "DLMM pair". An example is a document that compares
     two modules.
   - If two scopes use two words for one object, select one word. The code
     can use the words of an external source. Then keep the word of each
     scope, and give its meaning with a term of a larger scope.
   - Do not make a term of an approved word with the same meaning. Remove a
     term that no file uses (`stelint dictionary usage`).
8. Give each term of a dictionary one row of a table
   `| Term | Meaning |`. Write a verb with "to", for example "to walk".
9. Write the meaning of a term in words. Do not refer to an item of the
   code, because the name of the item comes from the term.
10. Use the type name as the term. The term "retry policy" is the type
    `RetryPolicy`. In a doc comment, write a link to the type if the
    language has links.
11. Write the full term each time: "source map", not "map".
12. Use the same word for the same object each time.
13. Put each code name in backticks. In a doc comment, write a link to
    each item that the documentation tool can find.
14. Do not use idioms, metaphors, slang or phrasal verbs ("set up",
    "look into").
15. Do not write a contraction, for example "don't" (`contraction`).
16. Do not write "e.g.", "i.e." or "etc." (`latin-abbreviation`). Write
    "for example", or write the full list.

## Sentences

17. Write a descriptive sentence with a maximum of 25 words. Write an
    instruction with a maximum of 20 words (`sentence-length`).
18. Write one fact or one instruction in each sentence.
19. Use the active voice. Identify the actor: "The parser gives the
    tokens", not "The tokens are given".
20. Use only the present tense, the past tense, the future tense, the
    imperative and the infinitive of a verb.
21. Do not add "-ing" to a verb (`ing-form`). A technical noun with "-ing",
    for example "heading", is permitted.
22. Keep the articles: "the", "a" and "an".
23. Write a noun cluster with a maximum of 3 nouns.
24. Use the present tense for the operation of the code. Use "must" only
    for an instruction to a caller or to a trait implementor.
25. Do not write a semicolon in a sentence (`semicolon`). Write two
    sentences.

## Paragraphs and format

26. Write a paragraph with a maximum of 6 sentences (`paragraph-length`).
    Write all its sentences about one object.
27. Put the most important information first. Put a condition before its
    instruction: "If the file does not exist, stop."
28. Use a list for 3 or more items. Use a table for data with two
    dimensions.
29. Wrap lines at 80 columns (`line-width`). Do not break a table row.

## Documentation comments

A module doc holds the documentation of the module of its file. An item
doc holds the documentation of the next item of the code. The writing
guide of each language gives its comments, for example `stelint-rust.md`.

30. A module doc starts with one sentence that tells the task of the
    module. Then write the callers of the module and the items that it
    calls.
31. An item doc starts with one sentence that identifies the item or tells
    its task. An index of the documentation shows only this sentence.
32. After the first sentence, write only the information that the
    signature does not give. This is the meaning of a value, an invariant,
    a necessary sequence or a side effect.
33. A field or a variant has one line: its contents and its invariant.
34. The doc of an item that is not public, and a comment that is not
    documentation, give the cause of the decisions in the code. Do not tell
    the operation of the next line.
35. `stelint-project.md` gives the sections of a module doc, if the project
    has a format for them. The overrides of `.stelint/config.toml` check
    them (`required-sections`, `section-order`).

## Text that a comment does not contain

36. Test names. Only the doc of a test can give the name of that test.
37. Build commands and test commands. Write them one time, in the document
    that tells how to build the project.
38. Dependency lists. The package manifest gives them, for example
    `Cargo.toml`.
39. Information about previous versions: no "was", "no longer" or "now".
    Commit messages hold this information.
40. The internal operation of a different module. Write a link to
    it.
41. The number of an instruction in a different file. Give the
    instruction in a small number of words. Give the name of its file and
    the heading of its section.
42. The text of ASD-STE100. Give only the number of its rule as a
    reference: "ASD-STE100 Issue 9, 6.3".

## Correct facts

43. Write each fact one time, at its source. Write a link to it from other
    text.
44. A comment that does not agree with the code is a bug. Change the
    comment, or change the code if the comment is correct, in the same
    change.
