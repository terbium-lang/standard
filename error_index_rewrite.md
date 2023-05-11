---
description: >-
  This document refers to all warnings and errors that could be thrown when
  trying to compile Terbium code (rewrite branch).
---

# Terbium Analyzer Warning/Error Code Index

## Warnings

Warnings are able to be ignored, and do not halt execution or further processing of your code. However, it is recommended to treat warnings as errors and only 
ignore them if you really have to.

## Errors

Error analyzers cannot be disabled and halt execution or further processing of your code.

### E000

There was an error trying to tokenize your source code. When Terbium wants to perform lexical analysis on your code, it will attempt to break the raw string of code into usable tokens with lexical meaning. Whenever an error occurs that prevents the tokenizer from tokenizing any further, this error is thrown by the compiler.

You may receive one of following messages in the error report:
 
 - *unexpected character 'X'*: Unexpected character found. The tokenizer only accepts a certain set of symbols and characters in your code, otherwise it must be hidden away in a comment, string, or raw identifier.
 - *unexpected newline in raw identifier*: A raw identifier, e.g. ``` `raw ident` ``` cannot span multiple lines. You should any newlines found inside the raw identifier.
 - *invalid escape sequence in raw identifier: 'X', expected \\ or \`*: Raw identifiers only accept two escape sequences: ``\\``, which transforms into a single backslash, and ``` \` ```, which transforms into a backtick.
 
