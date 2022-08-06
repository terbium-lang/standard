# Terbium Match Statements
This proposes an initial implementation for match expressions and statements which can match an object against multiple patterns.

## Proposal Details
Status: Implementation in progress  
Proposed for: Terbium Pre-release

## Summary
This proposal will add basic match statements like the following:

```ts
let n = 1;

let message = match n {
     0 -> "nothing",
     1 -> "one",
     2 | 3 | 4 -> "a few",
     else "a lot",
};
```

Match statements although called statements will act as expressions.

## Grammar
This proposal adds the following grammar rules:

```
PATTERN: <described below>
MATCH_OUTCOME: EXPR | BLOCK
MATCH_ARM: PATTERN "->" MATCH_OUTCOME
MATCH: "match" EXPR "{" (( MATCH_ARM "," )* ( MATCH_ARM | ("else" MATCH_OUTCOME) ) ","?)? "}"

...
EXPR: ... | MATCH
```

