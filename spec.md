---
description: >-
  This page goes over basic common language specifications for the Terbium
  Programming Language.
---

# Terbium Language Specification

This page goes over basic common language specifications for the Terbium Programming Language.

## Pre-release

Terbium's syntax is most "accurately" highlighted by existing TypeScript highlighters. Until syntax highlighting is commonly supported for this language (or where it has not been added), `ts` can be used as a replacement.

Many keywords/identifiers won't be highlighted, i.e. `func`.

## Runtime

The runtime execution of a Terbium program should go in the following order:

1. execute static blocks.
2. execute the entrypoint function.
3. execute cleanup tasks.

### Static Blocks

A *static block* is a block of code ran at the beginning of every imported module, including the source file. 
All top-level dynamic code are considered static blocks. A static block may also be a block labelled with `static`:

```ts
// Top-level code
let x = 1;
let y = 2;
println(x + y);

// Static block
static {
    let x = 1;
    let y = 2;
    println(x + y);
}
```

#### Lifetime of Static Blocks

A big difference between code executed in static-blocks is that although they are executed statically, **the scope of a static block
is dropped before calling the entrypoint.** That is, you will *not* have access to any local variables (i.e. `let`-defined variables)
within your main code:

```ts
let x = 1;

func main() {
    println(x); // Error! variable `x` was dropped prior to calling `main()`.
}
```

Think of top-level code being implicitly being wrapped with `static {}`:

```ts
// The above code should be interpreted as:
static {
    let x = 1;
} // `x` dropped here

func main() {
    println(x); // Error! variable `x` was dropped prior top calling `main()`.
}
```

Note that this is different from *static variables*, which exist throughout the runtime of the program:

```ts
static X = 1;

func main() {
    println(X); // Valid
}
```

### Entrypoint

The *entrypoint* of your program is where the runtime logic of your code generally *enters*. Techinically, code in static blocks
are ran before the entrypoint, however static blocks are ran regardless of whether they are directly executed from a binary or imported as a library
or module – this is in contrast to the entrypoint, which is never ran when importing a module or library, but only when directly executed from
a binary.

The entrypoint is specified as a **function** and is resolved in the following priority:
1. if there is a top-level function decorated with the `@entrypoint` decorator, use that function
2. if there is a top-level function named `main`, use that function
3. if strategies 1 and 2 fail, do not resolve an entrypoint function

If an entrypoint function exists, its signature will be checked against the following type:
```ts
trait MaybeError;
extend<T: Error> MaybeError for T;
extend MaybeError for never;

type Entrypoint = contained (args?: [string]) -> throws E
where
    E: MaybeError; // as a sum type (union): Error | never
```

That is:
- the function must take either zero parameters or one parameter of type `[string]`, preferrably named `args`
- the function must have a return type of either:
  - `void` (via `void throws E` when `E = never`, since `void throws never` flattens to just `void`), or
  - `void throws E` (shortened to just `throws E`), when you want error handling an dpropagation in the entrypoint function
- the function must be *contained*, which means it takes no captures. since the entrypoint function is assumed to at the top-level,
  such a scenario is impossible, anyways.

The following lints/errors regarding entrypoints should be issued:
- `no_entrypoint` when there is no entrypoint function to a binary (`warn` by default)
- `main_not_func` when there is no `@entrypoint` but there exists an item `main` that is not a function at the top-level (`warn` by default)
- `misplaced_entrypoint` when a function decorated with `@entrypoint` is not in the top-level (always error)
- `invalid_entrypoint` when an entrypoint function has captures or has an invalid signature (always error, see below) 

The following will resolve as entrypoint functions:

```ts
func main() {}
func main() -> throws Error {}

// With args
func main(args: [string]) {}
func main(args: [string]) -> throws Error {}

// With entrypoint
@entrypoint
func my_function() {}
```

The following demonstrates the priority `@entrypoint` has over `func main()`:

```ts
@entrypoint
func my_function() {} // <- `my_function` is the entrypoint function

func main() {} // `main` is NOT the entrypoint function
```

The following will **not** resolve as entrypoints:

```ts
const main = 5; // 5 is not a top-level function
// main_not_func: main is not a function
```

```ts
{
    func main() {} // `main` function is not at the top-level
} // `main` can no longer be accessed past this point as per scoping rules
```

```ts
// All of these functions take invalid parameters or return invalid types 
func main(x: int) {} // entrypoint function only takes either zero parameters
                     // or a single parameter of type `[string]` (typically named `args`)
func main() -> int {} // entrypoint function can only return either `void` or `void throws E` where `E: Error`
```

### Optional Parameters

`main` can optionally take a single parameter, `args` it is a `string[]` of the arguments passed to the `terbium` command.

```ts
// terbium main.trb arg

func main(args) {
    println(args); // ['main.trb', 'arg']
}
```

## Primitive types

Terbium comes with many primitive **types**. A type is the classification of an object.

### Number types

Numbers can be represented in memory in many different formats in Terbium.

| Type               | Description                                                              |
| ------------------ | ------------------------------------------------------------------------ |
| `uint`             | Unsigned integer, bit-width inferred                                     |
| `int`              | Signed integer, bit-width inferred                                       |
| `uintN` \[1]       | Unsigned N-bit integer                                                   |
| `intN` \[1]        | Signed N-bit integer                                                     |
| `char`             | Equivalent to a `uint32` that represents a single Unicode character \[2] |
| `float128`         | IEEE 754 floating point number, quadruple precision (128 bits)           |
| `float`, `float64` | IEEE 754 floating point number, double precision (64 bits)               |
| `float32`          | IEEE 754 floating point number, single precision (32 bits)               |

\[1] Valid values for N: `8`, `16`, `32`, `64`, `128`. \[2] See the [character literals](spec.md#character-literals) section for more information.

### Integer literals

An integer can be defined by simply writing the integer. By default, integer literals are defined as `int` (signed integers). Suffix the integer with `u` to define it as `uint` (unsigned).

```ts
5 // int
5u // uint
```

The cast syntax can be used to cast to specific bit-widths:

```ts
5::uint8
5u::int8
```

If casting fails, an error is raised:

```ts
256::uint8 // Max uint8 is 255, this will raise an error
```

If you want integers to wrap or coerce silently, use the `.wrapping_cast` method:

```ts
256.wrapping_cast<uint8>() // 255

// Using type-inference:
let x: uint8 = 256.wrapping_cast();
```

Integer literals can also be defined using radix specifiers:

| Prefix | Description | Example    |
| ------ | ----------- | ---------- |
| `0b`   | Binary      | `0b1101`   |
| `0o`   | Octal       | `0o167`    |
| `0x`   | Hexadecimal | `0x41ce3a` |

### Floating-point number literals

A floating-point number can be defined by simply writing the number with a decimal point. By default, float-literals are defined as `float` (`float64`).

```ts
5 // no decimal, int
5.0 // add .0 to make it a float
```

A trailing dot, either start or end, may be allowed or disallowed with varying implementations. Similiarly, multiple preceding zeros for floats (e.g. `00005.1`) and also integers (e.g. `00005`) may also be allowed or disallowed with varying implementations.

Floating-point numbers have their bit-widths specified through casting, similar to integers:

```ts
5.0::float32
```

### Booleans

A boolean is represented as the `bool` type. It can either be `true` or `false`. One can be created literally by writing `true` or `false`.

### Arrays

An array is a statically-sized, stack-allocated, and ordered collection of objects. Arrays cannot grow or shrink and their size must be known at compile-time.

The type of arrays are `T[N]`, where `T` is the type of an element in the array, and `N` is the size of the array. For example, `int[10]` is a 10-element array of integers. It's size, given `int` is resolved as a `int32`, is 32 \* 10 = 320 bits, or 40 bytes.

An array literal is defined through surrounding the elements with `[]`:

```ts
let my_array = [1, 2, 3, 4, 5]; // type inferred as int32[5]
```

### Slices

A slice is a view of an array in which the length of the slice may not be known at compile-time. The source array must exist and all values in the slice are borrowed values from the source array.

The type of slices are `T[]`, where `T` is the type of an element in the slice.

A slice cannot be defined literally since its data is borrowed from a source array. A slice may be retrieved by slicing an array, however:

```ts
let my_array = [1, 2, 3, 4, 5];
let first_two = my_array[..2]; // type inferred as int32[]
// contents of first_two are [1, 2]
```

#### Byte-slices

One type of slice (`uint8[]`) is a slice of `uint8`, better known as byte-slices.

### Tuples

A tuple is an ordered collection of objects of varying types (which are known), packed into one object.

The type of tuples are `(T1, T2, ...)`, i.e. a comma-separated list of types surrounded by `()`. For example, the tuple `(1, 2)` has the type `(uint32, uint32)`.

A tuple literal is defined by comma-separating its elements then surrounding them with `()`:

```ts
let my_tuple = (1, "string"); // type inferred as (int32, string)
```

Tuples may not grow or shrink, and the types of their elements cannot change.

### Lists

A `List` is a built-in collection type, however it is not considered a primitive type.

These are similar to arrays in the sense that they store ordered collections of objects, however these are heap-allocated and are growable and shrinkable. Their true size is only known at runtime.

A list is defined as follows:

```ts
struct List<T> {
    _ptr: RawPtr<T>, // raw pointer to the first element in the array
    _len: uint32, // the number of elements in the array
}
```

A list can be created literally by casting an array literal to a `List`:

```ts
let my_list = [1, 2, 3, 4, 5]::List; // type inferred as List<int32>
```

It can also be created by directly using its constructor:

```ts
let my_list = List(1, 2, 3, 4, 5); // type inferred as List<int32>
```

### Range

A `Range<Idx>` represents a range, which can store lower and upper bounds. These can represent ranges of numbers, characters, etc.

A range literal can be one of the following:

```ts
.. // Full range
n.. // n onwards, including n
..n // until n, but excluding n
..=n // until n, including n
m..n // m (inclusive) to n (exclusive)
m..=n // m (inclusive) to n (inclusive)
```

## Strings

A `string` should be stored as a raw `uint8[]` with a specified encoding. A `string` cannot be left without an encoding. By default, all strings will be in `utf-8` encoding. Strings are immutable and cannot grow nor shrink.

Since a `string` is stored as a slice, there must be a source `List<uint8>` the string is sliced from, or it must be from a string-literal or statically-created string. Usually, this is taken care of internally.

### String Literals

A string can be defined by surrounding the contents of the string with either `'` or `"`. By default, string literals will be encoded in `utf-8`.

```ts
'Single quoted string'
"Double quoted string"
```

#### Multi-line string literals

Multiline strings can be formed by surrounding the contents of the string with `#"` and its mirror counterpart (`"#`).

```ts
#"This string
spans
multiple
lines."#
```

You may add any N number of pound symbols, in which the string will be terminated by a quote followed by said N number of pound symbols.

```ts
##" // begin with ##
You can define a multi-line string in Terbium like this:
#"Hello world"#
"## // end with ##
```

#### Backslash escapes

In a normal string, backslash escape sequences exist for providing characters that were previously unnecessary to type out or impossible.

| Sequence          | Description                           |
| ----------------- | ------------------------------------- |
| `\n`              | Newline                               |
| `\r`              | Carriage Return                       |
| `\t`              | Tab                                   |
| `\b`              | Backspace                             |
| `\f`              | Form Feed                             |
| `\\`              | Literal backslash                     |
| `\'`              | Literal `'`                           |
| `\"`              | Literal `"`                           |
| `\0` \[1]         | Null byte                             |
| `\x12` \[2]       | Character by hex codepoint (2 digits) |
| `\u1234` \[2]     | Character by hex codepoint (4 digits) |
| `\U12345678` \[2] | Character by hex codepoint (8 digits) |

\[1] Because strings will **always** have an encoding, null-bytes can only be placed in byte-string literals. \[2] Numbers are a placeholder of a valid hex value of the specified length, e.g. `\u200b`

```ts
'Here\'s a contraction with apostrophes'
"This is a \"quote\" with quotation marks"
'This string has\nnewlines'
'An em dash: \u2014'
```

#### Raw strings

Prefix a string literal with `~` to make it a raw string. A raw string will not take into account backslash escapes.

```ts
~'Here is a raw string. A newline is represented as \n'
~#'
  A raw
  multiline string
  this is
'#
```

#### Interpolated strings

Prefix a string literal with `$` to add string-interpolation support to it.

```ts
world = 'World';
$'Hello, {world}!' // Hello, World!
```

In reality, string-interpolation is syntax sugar. The above roughly desugars to:

```ts
world = 'World';
'Hello, ' + world to string + '!'
```

See \[String Formatting] for more information.

#### Byte Strings

Strings without encodings are simply represented as byte slices: `uint8[]`. It is simply a string of bytes.

A literal string can be defined as a byte slice by adding `b` before the string:

```ts
b'Null byte: \0' // [78, 117, 108, ..., 58, 32, 0]
```

#### Character-literals

A `char` represents a single Unicode character. It is a "Unicode scalar value", and represented as a `uint32`.

A character-literal is represented by prefixing a single-character string-literal with a `c`:

```ts
let a = c'a';
```

You can also cast a one-character string to a `char`:

```ts
let a = "a" to char; 
```

Since `char` is internally represented as a `uint32`, you can also cast integers to chars as well:

```ts
let zws = 0x200b to char;
```

## Conditional expressions

A conditional expression is an expression that runs code depending on whether a boolean condition is either _true_ or _false_.

There are many conditional operators:

| Operator | Description                                     | Operation                                  | Trait Equivalent |
| -------- | ----------------------------------------------- | ------------------------------------------ | ---------------- |
| `==`     | Equals                                          | `op func eq(self, other: Rhs) -> bool`     | `Eq<Rhs = Self>` |
| `!=`     | Not equals                                      | `op func ne(self, other: Rhs) -> bool`     | `Ne<Rhs = Self>` |
| `<`      | Less than                                       | `op func lt(self, other: Rhs) -> bool`     | `Lt<Rhs = Self>` |
| `<=`     | Less than or equals                             | `op func le(self, other: Rhs) -> bool`     | `Le<Rhs = Self>` |
| `>`      | Greater than                                    | `op func gt(self, other: Rhs) -> bool`     | `Gt<Rhs = Self>` |
| `>=`     | Greater than or equals                          | `op func ge(self, other: Rhs) -> bool`     | `Ge<Rhs = Self>` |
| `x is T` | Check if the type of `x` is compatible with `T` | N/A                                        | N/A              |
| `x in y` | Contains                                        | `op func contains(self, value: V) -> bool` | `Contains<V>`    |

There are also three logical operators:

| Operator | Type   | Description |
| -------- | ------ | ----------- |
| `\|\|`   | Infix  | Infix       |
| `&&`     | Infix  | Logical AND |
| `!`      | Prefix | Logical NOT |

💡 _When an object `obj` is **truthy**, it means that `obj to bool == true`._

These logical operators also work with traditional values:

| Example | Action | 
| - | - |
| `a \|\| b` | Return a if a is truthy, else return b | 
| `a && b` | Return b if a is truthy, else return a |
| `!a` | Performs `op func not(self) -> Output` (trait equivalent is `Not<Output = Self>`). |

The operators `a || b` and `a && b` **short-circuit**, meaning while `a` is always evaluated as the main condition check, `b` is only evaluated when the value has to resolve to `b`. For example:

```go
func part_a() -> int {
    print("a");
    1
}
func part_b() -> int {
    print("b");
    1
}
func main() {
    // part_a() is called, which returns a truthy value. This means the left-hand value is returned,
    // and the right-hand value is discarded without ever being evaluated, so this line only prints
    // "a", and not "ab".
    part_a() /* truthy */ || part_b(); // a

    println(); // Print a line in between

    // part_a() is a truthy value, however with logical AND, the right-hand value is returned
    // if the left-hand value is truthy. This means part_b() is indeed called and "ab" is printed
    // on the next line.
    part_a() && part_b(); // ab
}
```

It should be noted that the logical operators `||` and `&&` only work if the left-hand value implements a cast function to a boolean. See the [casting](spec.md#type-casting) section for more information. Additionally, the types of both sides of the operator must be compatible, and the resulting type will be the _broader_ type of the two values. For example:

```go
struct A;
extend A {
    cast func to_bool(self) = true;
    func foo(self) = "a";
}
struct B : A;
extend A for B {
    func foo(self) = "b";
}

let a = A {};
let b = B {};
let result = b || a; // resulting type is A because A is broader than B, 
                     // even though we know mentally that this will be B
println(result.foo()); // prints "a"
```

### If-statements

The standard `if` statement can be used to run code if a condition is `true`:

```ts
let x = 1;
if x == 1 {
    println("x is 1");
}
```

The condition **must** be a `bool` and will not be implicitly casted to one:

```ts
let truthy = 1;
if truthy { ... } // does not compile

// Do this instead:
if truthy::bool { ... }
// or this:
if truthy == 1 { ... }
```

Use `else` to run code if a condition is false:

```ts
let x = 2;
if x == 1 {
    println("x is 1");
} else {
    println("x is 2");
}
```

The `else if` construct is also provided:

```ts
let x = 2;
if x == 1 {
    println("x is 1");
} else if x == 2 {
    println("x is 2");
} else {
    println("x is something else");
}
```

#### If-statements as expressions

If-statements that diverge can be used as expressions. If an if-statement is _divergent_, it means that code inside the if-statement will always be run, i.e. an if-statement with an `else`. The implicit-return syntax can be used to specify the output of if-statements:

```ts
let message = if x == 1 {
    "x is 1"
} else {
    "x is 2"
};
println(message);
```

The `then` keyword can make your code cleaner by removing the need for curly brackets:

```ts
let message = if x == 1 then "x is 1" else "x is 2";
println(message);
```

`else if` also works:

```ts
let message = if x == 1 {
    "x is 1"
} else if x == 2 {
    "x is 2"
} else {
    "x is something else"
};
// with the `then` keyword:
let message = if x == 1 then "x is 1"
    else if x == 2 then "x is 2"
    else "x is something else";
println(message);
```

Note that the `then` style of writing if-expressions **must** have an `else` block (i.e. it must diverge).

For consistency, you cannot mix `then` blocks with normal blocks, i.e.:
```ts
if x == 1 then 5 else {
    println("nope");
} // Invalid expression
```
...is an invalid expression.

### While-loops

A loop runs a block of code over and over again until it is told to exit. One type of loop is a _while-loop_, which, given a condition, will continuously run the condition until the condition is `false`.

The while-loop construct is found in most other programming languages and the concept is exactly the same:

```ts
let x = 0;
while x < 10 {
    print(x, " ");
    x += 1;
}
// 0 1 2 3 4 5 6 7 8 9
```

Control flow with `continue` and `break` is also provided to exit out of loops early:

```ts
// Above loop can be rewritten as:
let mut x = 0;
while true {
    print(x, " ");
    x += 1;
    if x >= 10 {
        break;
    }
}
```

#### `break if`, `continue if`

These can be used instead of a traditional if-statement as a shorthand to avoid another block and another level of indentation if following proper code styles. The above code can be rewritten as:

```ts
let mut x = 0;
while true {
    print(x, " ");
    x += 1;
    break if x >= 10;
}
```

#### `while-else`

An `else` block can be added to a while-loop, which will be run if the while loop was exited without a `break`:

```ts
let mut x = 0;
while x < 10 {
    x += 1;
} else {
    println("foo");
}
```

#### While-else expressions, breaking with values

While-else-loops can also be expressions. Since normal while-loops cannot be guaranteed to diverge, they are not considered expressions. `while-else` loops are always divergent.

Specify a value after `break` to break out of the while-loop with the value. For example:

```ts
let x = while true {
    break 1;
} 
// else-block is always needed for while-loops to convert them into experssions
else { 0 };

println(x); // 1
```

The `break if` grammar can be extended to `break [expression] if <condition>` to return values:

```ts
let mut x = 0;
let y = while true {
    x += 1;
    break 1 if x >= 10;
} else { 0 };
```

**If you ever break with a value, the type of all break-values in the same loop **_**must**_** be compatible with each other!** The type of the value returned from the while-loop will be the _broadest_ of all values. In the list of break-values, this includes the type of the value in the `else` block.

#### Loop statements

A `while true` loop can either run infinitely or break. If a `while true` loop is ever exited, it has diverged through a `break` statement. In this manner, `while true` loops do _not_ need an `else` block, since it will never be called.

A special case for this scenario would be inconsistent -- should it be resolved syntactially? Analytically? At runtime? Because of this, a more explicit type of loop is provided, inspired by the Rust Programming Language, the `loop`...loop:

```ts
// Rewrite the same code as above
let mut x = 0;
let y = loop {
    x += 1;
    break 1 if x >= 10;
};
```

A loop-statement:

* Logically the same as a `while true` loop
* Cannot take an `else` block
* Can always be an expression

## `when` and `match` statements

Chaining many `else if` together can be repetitive and make your code look bloated. Terbium provides the `when` statement for this purpose.

A `when` statement maps conditions to the expected result:
```ts
let x = 3;
when {
    x == 0 -> println("x is 0"),
    x > 0 -> println("x is positive"),
    // The else clause is optional
    else println("x is negative"),
}
```

The above is equivalent to:
```ts
let x = 3;
if x == 0 {
    println("x is 0");
} else if x > 0 {
    println("x is positive");
} else {
    println("x is negative");
}
```

Diverging `when`-statements can also be used as expressions. When used as expressions, the **else** clause must always exist.

```ts
let x = 3;
let sign = when {
    x == 0 -> 0,
    x > 0 -> 1,
    // When used as expressions, the else clause is required
    else -1,
};
```

You can also use blocks of code instead of just an expression in each arm:
```ts
let sign = when {
    x == 0 -> {
        println("x is 0");
        0
    },
    x > 0 -> {
        println("x is positive");
        1
    },
    else {
        println("x is negative");
        -1
    },
}
```

### `match` statements

The `when` statement is a powerful tool when dealing with many conditions in your code. However, there are times when a `match`-statement can help simplify your code even further.

A `match` statement is similar to a `when` statement, however it maps patterns rather than conditions to corresponding values. The patterns are matched against a subject value.

Take this `when`-statement, for example:
```ts
let x = 3;
when {
    x == 0 -> println("x is 0"),
    x == 1 -> println("x is 1"),
    x == 2 -> println("x is 2"),
    x == 3 -> println("x is 3"),
    x == 4 -> println("x is 4"),
    else println("x is something else"),
}
```

Even with a `when`-statement, this code still seems repetitive. This is why pattern-matching with `if`-statements is supported:
```ts
let x = 3;
match x {
    0 -> println("x is 0"),
    1 -> println("x is 1"),
    2 -> println("x is 2"),
    3 -> println("x is 3"),
    4 -> println("x is 4"),
    // If you're coming from a language from Rust, match-statements
    // do not necessarily have to diverge, so you do not have to exhaust
    // all possible patterns in a match-statement.
    //
    // Note that match-EXPRESSIONS are still required to diverge, however.
    else println("x is something else"),
}
```

## Functions

You can abstract a procedure a function which can be called over and over again.

A function may be defined with the `func` keyword:

```ts
func hello_world() {
    println("Hello, world!");
}
```

Then, it may be called by referencing the function (by its name in this case) followed by a call to the function using parenthesis:

```ts
func main() {
    hello_world(); // Call it once
    hello_world(); // Call it again
}
```

Functions may take **parameters**. Parameters are comma-separated and must have their type specified with a colon followed by the type:

```ts
func print_sum(x: uint, y: uint) {
    println($"{x} + {y} = {x + y}");
}

func main() {
    // Pass in 1 and 2 as arguments
    print_sum(1, 2); // 1 + 2 = 3
}
```

When calling functions that take parameters, all parameters without a default must be provided, otherwise a compile-time error is thrown. All types must also be met.

Function may also return values. The `return` keyword can be used, or an implicit-keyword can be issued by removing the semicolon off of the last expression of the function. Return types must be explicitly specified (for block-style functions) by specifiying the type after `->`.

```ts
// Explicit return
func sum(x: uint, y: uint) -> uint {
    return x + y;
}

// Implicit return
func sum(x: uint, y: uint) -> uint {
    x + y // Notice the lack of a semicolon
}
```

The return type for returning nothing is `void`, i.e. `func x() -> void { ... }`. If no return type is specified in the signature, the `void` type is used instead.

Functions may also be defined with the expression-style `=` shorthand, like so:

```ts
// Exact same as the functions above
func sum(x: uint, y: uint) = x + y;
```

In expression-style functions, the return type can be inferred and does not have to be explicitly specified.

The type of functions, and any callable, is written as `(parameters) -> return_type`. For example, the function defined above has the type `(uint, uint) -> uint`.

### Closures

A closure is a function that captures values from its outside scope. For example,

```ts
let x = 1;

func plus_one(val: int) = val + x; // x is taken from the outside scope
```

In the above function, values were implicitly captured. Values may also be explicitly captured with the `captures` keyword. Explicitly captured closures cannot use values that were not explcitly captured:

```ts
let x = 1;

func plus_two(val: int) captures x = val + x;
```

A function can be declared as `contained` to explicitly specify the function to capture nothing from outside scope. This means only variables created within the function, and static or constant variables, can be used in the function:

```ts
contained func plus_one(val: int) = val + 1; // no variables are captured here

// This throws a compile-error:
let x = 1;
contained func plus_one(val: int) = val + x;
```

### Anonymous functions

An anonymous function is a function declared without a name. The body of an anonymous function can be specified after an instance of the `do` keyword, and if the anonymous function
takes parameters, the `do` is prefixed with a backslash `\` followed by parameters:

```ts
// anonymous functions that take no parameters
let five = do 5;
println(five()); // 5

let print_five = do {
    println(5);
};
print_five(); // 5

// anonymous functions that take parameters
let double = \x do x * 2;
println(double(10)); // 20

let print_product = \a, b do {
    println(a, '*', b, '=', a * b);
};
print_product(5, 3); // 5 * 3 = 15
```

All parameters and the return type of anonymous functions can be inferred. All anonymous functions have inferred capturing of outside variables.

### Default values

If parameters are not passed into a function, Terbium will throw a compile-error. However, with default values, they are used instead and no error is thrown:

```ts
func add(x: uint, y: uint = 1) = x + y;

add(2) // 3
add(2, 2) // 4
```

Default values are lazily-evaluated expressions, so they can reference previous parameters:

```ts
func add(x: uint, y: uint = x) = x + y;

add(5) // 10
add(5, 1) // 6
```

The `default` operation is also implemented for many types. Placing a `?` after the parameter name makes the parameter default to the value returned by the default operation:

```ts
// default op implementation for uint:
extend uint {
    op func default() = 0;
}

func add(x: uint, y?: uint) = x + y; // y is default to 0

add(5) // 5
```

### Mutable parameters

Add `mut` before any parameter to make the parameter itself mutable:

```ts
func add(mut x: uint, y: uint) -> uint {
    x += 1;
    x + y
}

add(1, 1) // 3
```

### Keyword arguments

You may specify arguments to a function call by name using keyword arguments:

```ts
func add(x: int, y: int) = x + y;

add(x: 1, y: 1) // 2
```

Positional arguments may not come after keyword arguments:
```ts
add(x: 1, 1) // Syntax error!
```

## The Type System

The Terbium type system is a comprehensive type system which will be outlined in this section.

### Concrete types

Concrete types are types that are usable, constructible, and unabstract -- that is, structs, enums, classes, and not traits.

#### What is a `struct`?

A `struct` is the most basic way to define a concrete type. A struct has _fields_ in which their values are packed together in memory. For example:

```ts
struct Point {
    x: int,
    y: int,
}
```

Then, to create an instance of `Point`, use a _struct literal_:

```ts
let point = Point { x: 0, y: 0 };
```

You can use the `extend` keyword to add methods on `Point`. Let's declare the _construct_ operation so we can declare the struct via a constructor:

```ts
extend Point {
    op func construct(mut self, x: int, y: int) {
        self.x = x;
        self.y = y;
    }
}

// Create a point with a constructor
let point = Point(0, 0);
```

Note that any types that implement the _construct_ operation **cannot** be constructed using the struct-literal syntax. That means `Point { ... }` cannot be used to construct point anymore since we declared a constructor for `Point`, and will throw an error.

#### What is a `class`?

A `class` is a higher level way to define a `struct` and its methods, through a more familiar syntax. Here is the same `Point` class implemented as a class:

```ts
class Point {
    // Fields are defined here
    x: int;
    y: int;

    // Here is the constructor from before:
    op func construct(mut self, x: int, y: int) {
        self.x = x;
        self.y = y;
    }
}
```

Classes with fields _must_ implement the _construct_ operation in order to have a way to construct the class. No matter what, classes cannot be constructed with struct-literal syntax. Classes that have fields but do not have constructor implementations are called _stale classes_ since they cannot be instantiated.

**Desugaring classes**

`class` is essentially syntax sugar around `struct` and `extend`:

```ts
class MyClass {
    func method_a(self) {}
    func method_b() {}
}

// Desugar:
struct MyClass;
extend MyClass {
    // Classes always get a default constructor if one is not provided
    op func construct(self) {}
    // Classes always get a debug representation
    op func debug(self) -> string { /* implementation not shown */ }
    
    func method_a(self) {}
    func method_b() {}
}
```

Here is the same point class from before (with an extra method for completeness):

```ts
class Point {
    x: int;
    y: int;
    
    op func construct(mut self, x: int, y: int) {
        self.x = x;
        self.y = y;
    }

    /// Returns the distance of this point from (0, 0).
    func distance(self) = self.x.hypot(self.y);
}

// Desugar
struct Point {
    x: int,
    y: int,
}
extend Point {
    op func construct(mut self, x: int, y: int) { ... }
    op func debug(self) -> string { ... } // Automatically implemented

    /// Returns the distance of this point from (0, 0);
    func distance(self) = self.x.hypot(self.y);
}
```

#### What is an `enum`?

An enum is an enumeration of many _variants_ of an object represented as one value.

For example, an enumeration of colors:

```ts
enum Color {
    Red,
    Green,
    Blue,
}
```

The `Color` enum is seen to have 3 _variants_. It can only ever be in those three variants. If it is represented otherwise in memory, it is _undefined behavior_.

The _discriminant_ of enum variants is the value stored in memory that determines which variant an enum value is representing at compile time. Discriminants are automatically determined, however they can also be manually passed:

```ts
enum Color {
    Red = 0,
    Green,
    Blue,
}
```

By the default, the bit-width of the discriminant is automatically determined, based on the largest discriminant out of all variants. The smallest bit-width possible (unsigned) is used, however you can manually specify the representation with the `by` keyword:

```ts
enum Color by uint16 {
    Red,
    Green,
    Blue,
}
```

You can also inherit variants from other enums with a colon:

```ts
enum PrimaryColors {
    Red,
    Blue,
    Yellow,
}
enum Colors : PrimaryColors {
    // Red, Blue, and Yellow variants are inherited
    Green,
    Purple,
    Orange,
}
```

### Composition with traits

Traits are abstractions over classes that perform common behaviors. A particular type can have these behaviors and can share them with other types. Traits can be used to define shared behavior in an abstract way:

```ts
/// A trait for everything that has a name
trait Named {
    // No default implementation. This means all types that extend this trait must provide an implementation.
    func name(self) -> string;
    
    // Here we have a method with a default implementation
    func name_length(self) = self.name().len();
}
```

Types can implement traits with the `extend` keyword:

```ts
struct Human {
    first_name: string,
}
extend Named for Human {
    // Implement the required method
    func name(self) = self.first_name;
}

func main() {
    let human = Human { first_name: "Bob" };
    println(human.name()); // Bob
    println(human.name_length()); // 3
}
```

Classes have another way of implementing traits through the `with` keyword, mixing in the trait implementation with the general class declaration:

```ts
class Human with Named {
    first_name: string;

    op func construct(mut self, name: string) {
        self.name = name;
    }

    // Implement the required method
    func name(self) = self.first_name;
}
```

### Inheritance

Classes can inherit from parent concrete types. In this way, they inherit both the **fields** and their **methods**.

For example,

```ts
class Organism {
    classification: string;

    op func construct(mut self, classification: string) {
        self.classification = classification;
    }
}

// Use the colon to declare inheritance
class Human : Organism {
    name: string;

    op func construct(mut self, name: string) {
        // Call the super constructor
        super("human");
        self.name = name;
    }
}
```

### Union (sum) types

A union of types in Terbium represents a type that may represent one of many given types. A simple union type can be written as `A | B`.

Standard union types are safely represented as enums. This means they take up extra space in memory to store its enum **discriminant**:

```ts
type A = uint8[5]; // 5 bytes
type B = uint8[10]; // 10 bytes

type C = A | B;
// type C is represented as:
enum C {
    A(A), // 0 (discriminant) + (value of A) + 5 bytes of padding
    B(B), // 1 (discriminant) + (value of B)
}
// ...where the discriminant of the enum takes up an additional 1 byte.
// Therefore, the size of `C` is 1 + the size of the largest type, 10 = 11.
```

These are also typically referred to as _safe unions_.

#### 💡 Narrow and wide types

When a type is _narrowed_, it means that a _broader type_ is turned into one that is more _specific_.\
When a type is _widened_, it means that a _specific type_ is turned into one that is more _broad_.

For example, a coercion from `string` -> `uint8[]` is a type _widening_ since `string` is more specific than `uint8[]`. Similarly, a specification of a `uint` as a `uint8` is a type _narrowing_ since `uint`, which was broad over all unsigned integer types, has been narrowed into a more specified `uint8`.

When we union types together, we are _widening_ the types that are compatible with it. However, this _narrows_ the specificity of the value within the function.

For example, take the following scenario:

```ts
trait A {
    // Exclusive to A
    func a(self);
    
    func common(self) -> int;
}
trait B {
    // Exclusive to B
    func b(self);
    
    func common(self) -> string;
} 

func foo(x: A | B) = ...;
```

We can pass either values that meet `A` or `B` as `x`, since they are compatible by the union. However, when we want to use `x`, the type has been _narrowed_ by merging properties of both `A` and `B` into a single type under the hood.

```ts
// The A | B desugar
enum A_or_B { A(A), B(B) }
extend A_or_B {
    // Notice both a and b are gone, since there is no guarantee these methods can exist on A | B.
    
    // The common method remains, with its return type merged...as another union.
    func common(self) -> int | string {
        match self {
            Self.A(a) -> a.common(),
            Self.B(b) -> b.common(),
        }
    }
}
```

#### Unsafe unions: `RawUnion`

The `RawUnion` type provides a way to specify a union type that is represented _without_ the enum discriminant. At runtime, this makes the value unsafe to access since we cannot guarantee that the dynamically generated value is truly the type we think it is.

For example, in `A | B`, we can check at runtime if the value is `A` or `B` since we can access the enum discriminant of the value. However, in `RawUnion<A, B>`, we cannot, since there is no discriminant known.

#### Runtime union type coercion

For a given union type `A | B`, how would we check whether a type is `A` or `B` at runtime? How could we _widen_ the type into a more specific `A` or `B`?

The _check_ can be done with the `is` operator, which for `x is U`, given `x: T`, checks if `T` is compatible and more specific than `U`:

```ts
func a_or_b(x: A | B) {
    if x is A {
        println("A");
    } else {
        println("B");
    }
}
```

The _coercion_ can be done with a _cast_:

```ts
func a_or_b(x: A | B) {
    if x is A {
        println("A is ", (x to A).a());
    } else {
        println("B is ", (x to B).b());
    }
}
```

This is _checked_ with safe unions and will throw a runtime error if the cast cannot be performed. With `RawUnion`s, the cast will always succeed by a simple transmute, which could be **undefined behavior**!

### Product (and) types

_Union types_ are types that require a type to meet at least _one_ of the type constraints. The opposite would be _And types_, which require a type to meet _all_ type constraints, written as `A & B`.

Take the following function:

```ts
trait A {
    func a(self);
}
trait B {
    func b(self);
}
func a_and_b(x: A & B) {
    x.a();
    x.b();
}
```

Since we can guarantee `x` is both `A` and `B`, we can use methods from _both traits_. `&` is useful when making sure parameters meet _multiple_ type bounds or traits; not just one.

### Generics and Type bounds

When a type is **generic**, it means that the application could be generalized over any type. For example, the identity function (takes a parameter and returns it) does not have to be limited to one type:

```ts
// Provide all generic type names in between the angle brackets <>
func identity<T>(val: T) -> T {
    val
}
```

Here, we are saying "for any type `T`, this function will take a parameter of this type, `T`, and return a value of the same type `T`. For example, `T` could be substituted with `int`, making the signature `identity(val: int) -> int`.

#### 💡 Monomorphization

What was just described in the previous sentence is **monomorphization**. It turns all uncertain generic types into actual types. By looking at what types the function is generic over when it is called, Terbium can generic a separate function that operates for every type that is used.

For example:

```ts
// Before monomorphization:
func identity<T>(val: T) = val;
identity(1);
identity("hi");

// After monomorphization
func identity_uint32(val: uint32) = val;
func identity_string(val: string) = val;
identity_uint32(val);
identity_string(val);
```

#### Generics on types

Types which may contain data may be generic over the data it contains. One good example is the `List` type, in which the type of the elements in the `List` varies. This is why `List` takes a type parameter (e.g. `List<int32>`), which specifies the type of the elements in the list.

Here is a `struct` which is also generic over some type `T`:

```ts
struct Foo<T> {
    foo: T,
}
```

#### Type bounds

When being generic over any type, the type will be extremely narrow and we probably won't be able to do much about the type. This is why we may want to only be generic over types that meet a specific bound. Then, the type can be widened so that fields and methods that apply to that bound are usable.

Here, `T` is bound by `uint`. This means any type compatible with `uint` can be used in place of `T`, but nothing else:

```ts
// Use the T: Bound syntax
func add<T: uint>(x: T, y: T) -> T {
    x + y
}
```

Type bounds are commonly traits, for example:

```ts
// From the Named trait we defined above
func print_name_of<T: Named>(n: T) -> T {
    println(n.name());
    n
}
```

Join multiple trait bounds with `&`:

```ts
// Two sample traits
trait A { func a(); }
trait B { func b(); }

func a_and_b<T: A & B>(val: T) {
    val.a();
    val.b();
}

// (note that you could also do this)
func a_and_b(val: A & B) { ... }
```

#### The `where` clause

Additional type bounds can be added with the `where` clause, resembling that of the Rust Programming Language.

For example,

```ts
func foo<T>(val: T) 
where
    T: A, // T must extend A
    T: B, // T must extend B
{ ... }
```

This is the exact same as `<T: A & B>`.

Where clauses are not solely a different way to specify type bounds, however when type bounds are specified that way it may make type bounds more expressed and readable. Where clauses can also be used to bound types that are not directly type parameters of the immediate declaration.

For example:

```ts
trait Bar;

class Foo<T> {
    // Notice the type parameter T is declared outside of the class
    func foo() where T: Bar {}
}
```

It can also be used to bound the class itself:

```ts
class Foo<T> {
    func foo() where Self: Bar {} // Foo cannot directly access foo
}

class ExtendsFoo<T> : Foo<T> with Bar {} // ExtendsFoo can always access foo
```

## Metaprogramming: Macros and decorators

Metaprogramming is the concept that allows code to generate code. In this way, boilerplate and repetitive code can be reduced, and your code could be made more readable or provide a more elegant interface.

### Declarative macros

Declarative macros are substitution-like macros which match a token signature against provided tokens, and substitutes them into the given substitution. Declarative macros closely resemble declarative macros in the Rust Programming Language.

For example, replacing repetitive code:

```ts
macro my_macro {
    ($f:ident) -> {
        $f("Hello, world!");
    }
}

// Prefix with # to call it as a macro
#my_macro(println);
#my_macro(print);

// Macros expand to:
println("Hello, world!");
print("Hello, world!");
```

#### Token types

| Name             | Description                             |
| ---------------- | --------------------------------------- |
| `token`          | Any token                               |
| `ident`          | Any identifier, including soft keywords |
| `string_literal` | String literal                          |
| `int_literal`    | Integer literal                         |
| `float_literal`  | Float literal                           |
| `bool_literal`   | `true` or `false`                       |
| `literal`        | String, int, float, or bool literals    |
| `expr`           | Expression                              |
| `stmt`           | Statement                               |
| `block`          | Block of statements                     |
| `type`           | Type                                    |
| `vis`            | Visibility specifier                    |
| `pattern`        | Match pattern                           |
| `deco`           | Decorator, `@pt:`, or `@!pt:` specifier |
| `decl`           | Declaration of a function or type-like  |
| `path`           | Import path, i.e. `a.b.{c, d}`          |

You can also use `*` to match 0 or more of a token, `+` to match 1 or more, and `?` to match 0 or 1:

```ts
unhygenic macro foo {
    ($($idents:ident)+) -> {
        $(func $ident() { println("foo"); })+
    }
}

#foo(a b c);
a(); // foo
b(); // foo
c(); // foo
```

What's this `unhygienic` keyword? Let's talk about macro hygeine.

#### Macro hygeine

Macros by default are **hygenic**, in that all variables created inside of the macro are only accessible within the macro scope. For example,

```ts
macro foo {
    () -> {
        let x = 1;
    }
}

func main() {
    #foo();
    println(x); // compile-error, x is not defined
}

// main function expands to:
func main() {
    :{
        let x = 1;
    }
    println(x);
}
```

Therefore, if we want to leak the functions `a`, `b`, and `c` we defined above, we will have to use the `unhygienic` keyword to make the function able to leak variables defined within. This essentially "removes" the block scope from its expansion:

```ts
unhygenic macro foo {
    () -> { let x = 1; }
}

func main() {
    #foo();
    println(x); // 1
}
```

### Decorators

Decorators are annotations prefixed with `@` put on top of item declarations to modify their behavior.

#### Simple function decorators

A function-only decorator can be declared like a decorator in Python, which simply is a function that takes the decorated function as an argument and returns a new function. The decorator function is called at compile-time, so you will only have access to build-dependencies:

```ts
// Use decorator func to create a function-only decorator
decorator func one_more(f: () -> int) -> () -> int {
    func inner() -> int captures f {
        f() + 1
    }
    inner
}

@one_more
func sample() = 1;

func main() {
    println(sample()); // 2
}
```

Simple-function decorators can take parameters:

```ts
// This decorator takes the n parameter
decorator func n_more(f: () -> int, n: int) -> () -> int {
    func inner() -> int captures f, n {
        f() + n
    }
    inner
}

@n_more(2)
func sample() = 1;

func main() {
    println(sample()); // 3
}
```

The decorator will be called without parenthesis if no parameters are accepted after the initial function. If there are any parameters taken after the function, optional or not, the decorator will have to be called _with_ parenthesis.

#### Procedural decorators

A procedural decorator generates code using pure Terbium code. It takes the AST (Abstract Syntax Tree) of the function or item being decorated and you can transform and return back a new AST made from the source AST.

Use the `decorator` keyword to create a procedural decorator (not `decorator func`):

```ts
import core.ast.AstNode;

decorator foo {
    // Within here, you may only use build dependencies
    
    // Inside of a decorator declaration, `ast` is already
    // defined as the ast of the item being decorated. 
    ast // Return back the ast
}

@foo
func unchanged() { /* implementation not shown */ }
```

Procedural decorators can also take parameters:

```ts
// Notice the inclusion of a parameter list with "decorator foo(...)"
// instead of "decorator foo" means the decorator is called with
// parenthesis
decorator foo(x: int) {
    ... // x can be used within here
}

@foo(1)
func bar() { ... }
```
