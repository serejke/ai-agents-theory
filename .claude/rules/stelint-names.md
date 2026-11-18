<!--
stelint 0.3.0 wrote this file. Do not change it. `stelint update` writes
it again from the text of the installed stelint. Write the rules of this
project in stelint-project.md.
-->

# The names of the code

This file tells how to give a name to an item of the code. A crate, a
module, a type, a trait, a function and a field are items. stelint does not
check names. The writer obeys this file.

The names and the prose use the same words. The dictionaries of
`.stelint/dictionaries/` give the meaning of each term. Thus a reader finds
the meaning of a name in the dictionary, and the code and its comments use
one word for one object.

`stelint-project.md` gives the rules of this project. If a rule of
`stelint-project.md` does not agree with a rule of this file, obey
`stelint-project.md`.

## Names and the dictionaries

1. Make each name from the words of `stelint-prose.md`, "Words".
2. Give an item the term of the dictionary for its object: `RetryPolicy`
   for the term "retry policy".
3. If the object has no term, add the term and its meaning to a dictionary
   first. Then give the name.
4. Put a term in the dictionary with the smallest scope that holds all the
   code that uses it (`stelint-prose.md`, "Words").
5. Write the words of a name in the format of the language. For a type,
   write `RetryPolicy`. For a field, write `retry_policy`.
6. Use an abbreviation only if a dictionary or a word list gives it. Add an
   abbreviation to a dictionary only if the names with the full term are
   too long.

## The information in a name

A reader frequently sees a name in a different file from its definition.
Thus the name must identify the item without its definition.

7. A type tells the data that it holds: `RetryPolicy`, not `Policy`.
8. A trait tells the task of its implementor, as a noun: `Parser`.
9. A function tells its task, as a verb with its object: `parse_header`.
   A query without a side effect can be a noun: `span()`.
10. A boolean is a condition that is correct if the value is `true`:
    `is_approved`, `has_history`.
11. An enum variant tells one of the values of its enum. It can be a
    participle or an adjective, because the enum gives the noun:
    `OrderEvent::Filled`.
12. A module, a folder and a file tell the term of their contents.

## Words that a name does not contain

13. No metaphor. A name tells the task of the item. It does not compare
    the item with a different object: `EventRecorder`, not `Tap`.
14. No participle or adjective without a noun as a type name:
    `ParsedFile`, not `Parsed`.
15. No type name of one general noun: `Data`, `Info`, `Item`, `State`,
    `Context`, `Config` or `Result`. A term of the dictionary can be one
    word.
16. No word for a type of code: `Manager`, `Handler`, `Helper`, `Util`,
    `Processor`, `Impl` or `Wrapper`.
17. No time and no version: `New`, `Old`, `Legacy`, `V2` or `Tmp`. The
    commit messages tell about the previous versions.
18. No test word in code that is not test code: `Fake`, `Mock`, `Stub` or
    `Test`.
19. No name of the crate or the module again: `parser::Parser`, not
    `parser::ParserParser`.

## One word, one meaning

20. Give each word one meaning in each file: the meaning that a dictionary
    of the file gives.
21. If a dictionary gives the meaning of a verb, use that verb in a name
    only in that meaning.
22. Give each object one name. Do not give it a second name: `fee_payer`,
    and not also `wallet` or `payer`.
23. A value keeps its name while it keeps its type. A value of a different
    type gets a different name: a `Header` that the parser checks becomes a
    `CheckedHeader`.
24. Use the smallest number of words that make the name different from
    the other names of its module.
25. Give a name of one letter only to a type parameter, or to a closure
    argument of one line.

## A rename

26. Rename the item, its file, its documentation and its term of the
    dictionary in one change.
27. In the same change, change each text that holds the name: tests,
    snapshots, serialized names and log targets.
28. Do not keep the previous name as an alias.
