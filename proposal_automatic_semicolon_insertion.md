# Terbium Automatic Semicolon Insertion
Terbium Automatic Semicolon Insertion (Terbium **ASI**) is a feature in the Terbium compiler which eliminates the need to insert semicolons when writing code by "automatically" inserting semicolons where needed.

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

Nobody *actually* writes their code all on one line unless they want to obfuscate their code -- at least for ahead-of-time compiled languages where the source code is not shipped but rather the compiled binary, so it doesn't matter how large the source files are. Programmers will usually write at most *one statement per line*, and a statement can span *multiple lines*. A rule can be derived from this assumption: **ASI will *only* insert semicolons at newlines**.

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

#### Indentation

Good programmers will always indent their code to make it obvious where blocks start and end, and how deeply nested they are, although the spacing of these indents can vary. Similarly, when a statement spans multiple lines, or more specifically, the *expression part* of a statement span lines, indents are used as a visual indicator that the expression has not ended and spans multiple lines.

For example, a programmer may split a *method chain* over multiple lines if it gets too long:
```rust
let total: Array = (0..100)
    .map(x do x * 2)
    .step_by(2)
    .zip(0..)
    .map((x, y) do x * y)
    .filter(x do x < 20)
    .to()
```

Notice the indents following the initial line of the expression. Method chains could be ambiguous with the proposed [*implicit member* syntax](https://github.com/terbium-lang/spec) which allows the syntax `.attr` to be parsed as a valid expression, where the target of the attribute is implied through a system such as type inference. 

Let's say that one wanted to use this proposed syntax with a let-statement before it:
```ts
let x = 1
.attr
```
Again, Terbium has no idea that `1.attr` is invalid because ASI only occurs during the parsing stage. Neither does it know that `.attr` is *always* invalid (unless an implicit return) because its type is unknown. Therefore, it must disambiguate.

The first example involving method-chaining usually involves *indents* for every method. On the contrary, the second example involving the propsed implicit member syntax does not use indents, because the programmer **intended** for `.attr` to be its own expression. This ties back to intent. When method chaining, a programmer will usually *indent* chained methods, but when using the implicit member syntax, they will not indent because they *intend* for it to be a separate expression.

New rules can be derived from this knowledge:
- **When encountering any number of newlines followed by an indent with a larger indentation width than that of the previous line, do *not* insert a semicolon *if possible*.** In other words, only insert a semicolon if there is invalid syntax without it. 
- **When encountering newlines followed by an indent with a smaller or equal indentation width than that of the previous line, try inserting a semicolon.** In other words, a semicolon will be inserted regardless of whether the syntax is valid without them.

This makes these two statements differ in behavior:
```ts
// The expression is indented to show intention of attaching
// that expression to the `return` statement.
return
    super_long_expression_that_is_indented_on_a_separate_line() // ;
```
```ts
// The expression is not indented to show intention to
// separate the `return` statement from the expression
return // ;
super_long_expression_that_is_on_a_separate_line() // ;
```

This rule would *almost* work except for a few caveats:

Some programmers prefer to use spaces for indentation. Determining the indentation width from spaces is easy, just take the number of consecutive spaces before a non-whitespace token. Other programmers may prefer tabs, which is a bit more
challenging, since the width of tabs varies throughout editors and fonts. However, almost *all* programmers will choose to
*exclusively* use spaces or tabs, and never mix the two. Because of this, any whitespace, whether it be spaces or tabs, will
be considered to have a width of 1. A codebase that mixes spaces and tabs is quite rare, so the compiler will not take
into consideration normalizing tab widths. Two tabs will be considered the same indentation width as two spaces. To discourage mixing tabs and spaces, a `spaces_mixed_with_tabs` lint will be present in the compiler.

Finally, take the following method-chain:
```ts
let sample = func_with_lots_of_parameters(
    a: 1,
    b: 2,
    c: 3,
    d: 4,
)
.method()
```

Keeping the `.` in `.method()` the same level as the ending `)` is usually preferred over indenting `.method()`. However, because of the lack of the indent, the `.method()` part will be parsed as a standalone expression. This is a pending disambiguation, and since the implicit member syntax is still a proposal, it should not affect current Terbium code. The code above should still work as intended. A more realistic situation would be with binary operators, however:
```ts
let sample = func_with_lots_of_parameters(
    ...
)
+ func_with_a_very_long_name()
+ 1
```

In this case, both expressions at the end would be parsed as standalone unary-expressions.
There are a few workarounds to this (you can also use [backslashes](#backslashes)):

Although ugly, you can always indent the two expressions:
```ts
let sample = func_with_lots_of_parameters(
    ...
)
    + func_with_a_very_long_name()
    + 1
```

You could also wrap the chain in parenthesis:
```ts
let sample = (
    func_with_lots_of_parameters(
        ...
    )
    + func_with_a_very_long_name()
    + 1
)
```

You could also move the first operator to be on the same line as the closing `)`, then double-space indent:
```ts
let sample = func_with_lots_of_parameters(
    ...
) + func_with_a_very_long_name()
  + 1
```

You could move the operators to the end of its first operand to force invalid syntax with a semicolon insertion:
```ts
let sample = func_with_lots_of_parameters(
    ...
) + 
func_with_a_very_long_name() +
1
// Do notice that it is easy to miss the operators when skimming the code, so
// it is discouraged to use this workaround.
```

If none of the workarounds work, you may also turn off ASI.

### Backslashes

Backslashes are another way to resolve the ambiguity above. When a backslash is placed before a newline, ASI will ignore that newline and not consider inserting a semicolon there.

Therefore, the example above could be written as:
```ts
let sample = func_with_lots_of_parameters(
    ...
) \
+ func_with_a_very_long_name() \
+ 1
```

### General Constraints

Apart from rules that make ASI activated with intent, here are the *general* rules that ASI must follow:

- ASI will try inserting a semicolon at *every* newline which does *not* precede a backslash. **If the insertion of a semicolon renders the new syntax invalid, a semicolon will *not* be inserted.**
- **ASI will not insert a semicolon on implicit returns.** To void an implicit return, a semicolon must be explicitly written by the developer.

## Disabling ASI

Some programmers may prefer mandatory semicolons because even with all the "intent" magic, semicolons will always offer direct disambiguation between statements. Source code from a *package* can have ASI disabled by adding the following to `terbium.toml`:
```toml
[compiler]
asi = false
```
Other packages/dependencies be compiled based on the preference in their own `terbium.toml`.

If directly using `terbium compile` to compile, add the `--no-asi` flag:
```shell
$ terbium compile --no-asi main.tb
```
