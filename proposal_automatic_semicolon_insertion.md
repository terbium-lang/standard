# Terbium Automatic Semicolon Insertion
Terbium Automatic Semicolon Insertion (ASI) is a feature in the Terbium compiler which eliminates the need to insert semicolons when writing code by "automatically" inserting semicolons where needed.

## Motive
Terbium expects every statement to terminate with a semicolon, with only standalone braced-statements (e.g. braced if-statements, while-loops, blocks, and item definitions) not requiring one. *Implicit returns* are expressions at the end of blocks which do not terminate with a semicolon to indicate the block *resulting* in that expression.

For example, here is a function written in Terbium which computes the factorial of a given integer `n`:
```ts {1,4-5}
func factorial(mut n: uint) {
    let mut result = 1;
    while n > 1 {
        result *= n;
        n -= 1;
    }
    result
}
```

 At the end of lines 1, 4 and 5, semicolons are added in order to signify the end of statements. With ASI, the code could be rewritten as:
```ts {1,4-5}
func factorial(mut n: uint) {
    let mut result = 1
    while n > 1 {
        result *= n
        n -= 1
    }
    result
}
```

Simply eliminating the need for semicolons cleans up the code and makes it more "visually appealing" to read. It enables rapid prototyping, improves developer experience, and makes the learning experience of Terbium easier by removing a syntactic barrier from the language.

## Ambiguities
Many questions can be raised regarding ASI in Terbium. A main purpose of requiring semicolons is to reduce ambiguity and make it obvious and intentional, to both the programmer, those who will read the code, and the compiler itself, for when a statement ends and the next one starts.

### Binary or expression-then-unary?
Take the following block:
```ts
{
    let x = 1
    + 1
}
```
Semicolons could be inserted in one of *three* ways:
```ts
{
    let x = 1; // end of statement
    + 1; // end of statement
    // implied void as the result of the block
}
```
```ts
{
    let x = 1; // end of statement
    + 1 // no semicolon inserted here,
        // `+1` will be the result of the block through implicit return
}
```
```ts
{
    let x = 1 // no semicolon here
    + 1; // expression continues here
    // implied void as the result of the block
}
```
How should the compiler disambiguate this? The `let` statement could be interpreted as either 
- a let declaration and then a standalone expression (first and second examples),
- or include the `+ 1` with the `let` statement, since doing so still makes the statement valid (third example).

And with the former, another ambiguity stems out of Terbium's support for implicit returns. The standalone expression being at the end of the block means a semicolon does not have to be inserted for an implicit return. We will resolve this ambiguity further down in this document and just focus on the aforementioned ambiguity for now.

And even *if* the *compiler* could disambiguate, the ambiguity is not immediately obvious to a reader of the code, nor would it be obvious to those who write the code without proper tooling.

A reader would have to learn disambiguation rules, and a programmer who is not immediately clear about these rules may run into unexpected bugs in their code which are hard to debug -- in the example above a programmer may believe in their mind that `x` should be `1`, while another reading the code may believe that `x` should be `2`.

### Return values
Another ambiguity occurs when returning values. 

Take the following code:
```ts
return
1
```

This could again, be interpreted (mainly) in two ways:
```ts
return; // return void;
1;      // standalone expression
```
```ts
return
1; // return 1;
```

Now it immediately appears to a *human* that this code was *likely* designed to be written as one way or another if they know the supposed return type of the function. For example if they know the function should return `int`, the intended interpretation should be `return 1;`. 

Even then, a standalone `1` does nothing so a developer would likely consider the `return 1;` disambiguation.  But to the compiler, parsing and ASI occurs *before* type lowering and type inference, so there must be a consistent rule. And what if instead of `1`, the expression was maybe an effectful function call? Consider,
```ts
return
println("Hello, world!")
```
The call to `println` is an expression just like `1`, however `println` is a function that is *designed* to be standalone, since it returns `void`.

And again with debugging, the it is not immediately obvious what the code will do. Will it
- early return, preventing the call to `println` from ever being executed, or
- return the call to `println`, printing text to standard output?

## Disambiguation and Rules

All of these ambiguities can be solved by a common set of rules. However, these rules are designed to be what a programming would write based on **intent**. In fact, *intent* is the solution to the disambiguities a programmer or code reader would, in theory, face.

### How is intent achieved?

Terbium's ASI rules are designed to only insert semicolons where intended by the programmer. It does this through leveraging code conventions which are common to both *code formatters* and just common *code styling*. A good programmer will format their code in a way that reduces clutter and conveys *intent* to readers of their code.

Nobody *actually* writes their code all on one line unless they want to obfuscate their code -- at least for ahead-of-time compiled languages where the source code is not shipped but rather the compiled binary, so it doesn't matter how large the source files are. Programmers will usually write at most *one statement per line*, and a statement can span *multiple lines*. A rule can be derived from this assumption. **ASI will *only* insert semicolons at newlines**.

This means that an expression like this:
```ts
a() b()
```
Is still invalid syntax, even with ASI. It would have to be written with a semicolon:
```ts
a(); b()
```

However, in:
```ts
a()
b()
```
A semicolon will be considered for insertion because of the newline between `a()` and `b()`.

