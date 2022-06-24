# Terbium Analyzer Warning/Error Code Index
This document refers to all warnings and errors that could be thrown when trying to compile Terbium code.

## Warnings
Warnings are able to be ignored, and do not halt execution or further processing of your code.
However, it is recommended to treat warnings as errors and only ignore them if you really have to.

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
A variable was declared as `mut`, but it was never mutated

Don't declare a variable to be mutable if it isn't necessary:
```ts
let mut x = 1; // Bad
x // We never mutated x
```

Simply declare it as immutable, and make it mutable when you need to:
```ts
let x = 1; // Good
x
```

For reference, here are Terbium's mutability rules in a flash:
- Reassignment requires mutability
- Passing a variable to a parameter declared as `mut` requires mutability.
  Objects not assigned to a variable will get their mutability in the function's scope.
- Scope reservation (e.g. `let x; ... x = 1;`) does not require mutability

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
Error analyzers cannot be disabled and halt execution or further processing of your code.

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
my_vairable // Error, we made a typo!
my_variable // Simply fix the typo
```

### E002
A variable declared as `const` was redeclared.

Once you declare a variable as `const`, you may never redeclare a variable
or function with the same name in its same scope:
```ts
const x = 1;

func foo() {
    let x = 2; // Error, we already declared x as const
}
```

`let` is more lenient, although immutable, you can redeclare variables declared
with `let`, even in the same scope:
```ts
let x = 1;
let x = 2; // Works

func foo() {
    let x = 3; // Works
}
```

Let-redeclaration is usually done to persist immutability, to switch a variable's scope,
or reassign something of a different type to the same identifier.

### E003
An immutable variable was reassigned to.

Variable reassignment, that is assigning to a variable without using any `let` or `const`,
is only valid for mutable variables:
```ts
let x = 1;
x = 2; // Error, x was not declared as mutable

// do note...
let x = 3; // Works - note that redeclaration is allowed.
```

Declare a variable as `mut` to fix this:
```ts
let mut x = 1;
x = 2; // Works
```
