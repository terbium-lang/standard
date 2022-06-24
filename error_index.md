# Terbium Analyzer Warning/Error Code Index
This document refers to all warnings and errors that could be thrown when trying to compile Terbium code.

## Warnings

### W000
Non-type identifier names should be snake_case.

Other casings such as camelCase are discouraged:
```ts
let myVariable = 1; // Bad
```

Instead, try using snake_case instead:
```ts
let my_variable = 1; // Good
```

This allows for consistent and cleaner casing throughout the entire 
language and external libraries you may use.

### W001
Type identifier names, such as classes or traits should be PascalCase.

When declaring a type, such as a class, try not to use snake_case or any other casing:
```ts
class my_class { /* ... */ } // Bad
```

Instead, use PascalCase, or "UpperCamelCase" as it is known by some:
```ts
class MyClass { /* ... */ } // Good
```

This allows for easier disambiguation between which variables are
normal variables or functions, and which are types or classes.

### W002
Identifier names should contain only ASCII characters.

Although allowed, try not to use non-ascii characters in identifier names, such as variable names:
```ts
let 不是ascii = 1; // Bad
```

Instead, it is best to use characters that can be typed on a normal keyboard:
```ts
let surely_ascii = 1; // Good
```

This allows for you and others who use your code to easily reference variables
in your code.

### W003
A variable or paramter was declared but never used.

If you've declared a variable, chances are you want to use it
later on in your code.

Try not to declare unused variables or parameters:
```ts
let unused = 1; // Bad
// EOF

// or...
let [ used, unused ] = [1, 2]; // Bad
println(used);
// EOF

// or...
func f(a) { 1 } // Bad, parameter a is never used
```

If it is intentional to declare the unused variable, for example 
as an artifact of destructuring, or due to it being required
when passed as callback, prefix it with an underscore (`_`),
or name it `_` in its entirety:
```ts
let _ = 1; // Good
let _unused = 1; // Good

// or...
let [ used, _unused ] = [1, 2]; // Good
let [ used, _ ] = [1, 2]; // Good

// or...
func f(_a) { 1 } // Good
func f(_) { 1 } // Good
```

If a variable truly does not need to be declared, then don't declare it!

### W004
A variable was declared as `mut`, but it is never mutated

*explanation todo*

### W005
Global mutable variables are highly discouraged.

Try not to declare variables in the top level which are `mut`:
```ts
// START
let mut x = 1; // Bad
```

Instead, mutable variables should only be scoped in something
such as a function (like the `main` function):
```ts
func main() {
    let mut x = 1; // Good
}
```

Global mutable variables create something called "global mutable state".
These can lead to unknown or unwanted behaviors such as data races.

## Errors

### E000
Invalid syntax. Rather than being caught during analysis, this is caught during tokenization or parsing.

There are many cases of invalid syntax:
- Unexpected token, such as an operator right after another operator. In this scenario, the parser expects an expression instead.
- Unexpected end, such as an incomplete binary infix operator (e.g. `1 + /* EOF */`).
- Expected token. An example would be forgetting a semicolon after a variable declaration (`let x = 1`).
- Unclosed or unbalanced delimiters/brackets, such as an unclosed brace: `func main() { 0` In this case, simply close the delimiter.
  The error message should be smart enough to provide you with where and what you should insert.
- Encountered the `const mut` declaration. Historically, this syntax used to be valid and instead the error would be thrown
  by the analyzer. This is now a syntax error - replace `const mut` with `let mut` instead.

### E001
An identifier (e.g. a variable) could not be found in the current scope.

This may have been a typo, please check spelling carefully:
```ts
let my_variable = 1;
my_vairable // Bad, we made a typo!
my_variable // Simply fix the typo
```

*todo*
