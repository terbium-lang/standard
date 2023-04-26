
# Terbium Language Specification
This page goes over basic common language specifications for the Terbium Programming Language.

## Pre-release
Terbium's syntax is most "accurately" highlighted by existing TypeScript highlighters.
Until syntax highlighting is commonly supported for this language (or where it has not been added), `ts` can be used as a replacement.

Many keywords/identifiers won't be highlighted, i.e. `func`.

## Runtime
Although Terbium should be designed to work at the top-level, the convention is to put the main execution of the program in
a `main` top-level function.

`main` is never called if a program is evaluated through another program, i.e. through `require`.

### Resolution
`main` should be resolved if `main` is available at the top-level of the program and references a callable object.

It is not to be resolved if `main` has a deferred resolution, or if `main` is set dynamically.
It will also not be resolved if `main` is not top-level or if it is not in the global scope.

If `main` exists as a global variable but is not callable, a warning should be given.

The following will resolve:
```ts
func main() {} // main is a function
```
```ts
const main = () => {} // main is an anonymous function
```
```ts
// main is a class which is callable through its constructor
class main {
    op func construct(self) {}
}
```

The following will **not** resolve:
```ts
const main = 5; // 5 is not callable
// Warning: main is not a callable function
```
```ts
// Warning: main is not initialized
let mut main: () -> void;

func resolve_main() {
    main = () => {};
}
resolve_main();
```
```ts
:{
    func main() {}
} // main cannot accessed past here

// now, no main function exists
// no warning is emitted
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

| Type               | Description                          |
| ------------------ | ------------------------------------ |
| `uint`             | Unsigned integer, bit-width inferred |
| `int`              | Signed integer, bit-width inferred   | 
| `uintN` [1]        | Unsigned N-bit integer               |
| `intN` [1]         | Signed N-bit integer                 |
| `float128`         | IEEE 754 floating point number, quadruple precision (128 bits) |
| `float`, `float64` | IEEE 754 floating point number, double precision (64 bits)     |
| `float32`          | IEEE 754 floating point number, single precision (32 bits)     |

[1] Valid values for N: `8`, `16`, `32`, `64`, `128`.

### Integer literals
An integer can be defined by simply writing the integer.
By default, integer literals are defined as `int` (signed integers). Suffix the integer with `u` to define it as `uint` (unsigned).

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

### Floating-point number literals
A floating-point number can be defined by simply writing the number with a decimal point.
By default, float-literals are defined as `float` (`float64`).

```ts
5 // no decimal, int
5.0 // add .0 to make it a float
```

A trailing dot, either start or end, may be allowed or disallowed with varying implementations.
Similiarly, multiple preceding zeros for floats (e.g. `00005.1`) and also integers (e.g. `00005`) may also be allowed or
disallowed with varying implementations.

Floating-point numbers have their bit-widths specified through casting, similar to integers:
```ts
5.0::float32
```

### Booleans
A boolean is represented as the `bool` type. It can either be `true` or `false`. One can be created literally by writing
`true` or `false`.

### Arrays
An array is a statically-sized, stack-allocated, and ordered collection of objects. Arrays cannot grow or shrink and their
size must be known at compile-time.

The type of arrays are `T[N]`, where `T` is the type of an element in the array, and `N` is the size of the array.
For example, `int[10]` is a 10-element array of integers. It's size, given `int` is resolved as a `int32`, is 32 * 10 = 320 bits, or 40 bytes.

An array literal is defined through surrounding the elements with `[]`:
```ts
let my_array = [1, 2, 3, 4, 5]; // type inferred as int32[5]
```

### Slices
A slice is a view of an array in which the length of the slice may not be known at compile-time. The source array must exist
and all values in the slice are borrowed values from the source array.

The type of slices are `T[]`, where `T` is the type of an element in the slice.

A slice cannot be defined literally since its data is borrowed from a source array. A slice may be retrieved
by slicing an array, however:

```ts
let my_array = [1, 2, 3, 4, 5];
let first_two = my_array[..2]; // type inferred as int32[]
// contents of first_two are [1, 2]
```

### Tuples
A tuple is an ordered collection of objects of varying types (which are known), packed into one object.

The type of tuples are `(T1, T2, ...)`, i.e. a comma-separated list of types surrounded by `()`.
For example, the tuple `(1, 2)` has the type `(uint32, uint32)`.

A tuple literal is defined by comma-separating its elements then surrounding them with `()`:
```ts
let my_tuple = (1, "string"); // type inferred as (int32, string)
```

Tuples may not grow or shrink, and the types of their elements cannot change.

### Lists
A `List` is a built-in collection type, however it is not considered a primitive type.

These are similar to arrays in the sense that they store ordered collections of objects, however these are heap-allocated
and are growable and shrinkable. Their true size is only known at runtime.

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

#### Byte-slices
One type of slice is a slice of `uint8`, better known as byte-slices.

## Strings
A `string` should be stored as a raw `uint8[]` with a specified encoding. A `string` cannot be left without an encoding.
By default, all strings will be in `utf-8` encoding. Strings are immutable and cannot grow nor shrink.

Since a `string` is stored as a slice, there must be a source `List<uint8>` the string is sliced from,
or it must be from a string-literal or statically-created string. Usually, this is taken care of internally.

### String Literals
A string can be defined by surrounding the contents of the string with either `'` or `"`.
By default, string literals will be encoded in `utf-8`.

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

|     Sequence     |              Description              |
| ---------------- | ------------------------------------- |
| `\n`             | Newline                               |
| `\r`             | Carriage Return                       |
| `\t`             | Tab                                   |
| `\b`             | Backspace                             |
| `\f`             | Form Feed                             |
| `\\`             | Literal backslash                     |
| `\'`             | Literal `'`                           |
| `\"`             | Literal `"`                           |
| `\0` [1]         | Null byte                             |
| `\x12` [2]       | Character by hex codepoint (2 digits) |
| `\u1234` [2]     | Character by hex codepoint (4 digits) |
| `\U12345678` [2] | Character by hex codepoint (8 digits) |

[1] Because strings will **always** have an encoding, null-bytes can only be placed in byte-string literals.
[2] Numbers are a placeholder of a valid hex value of the specified length, e.g. `\u200b`

```ts
'Here\'s a contraction with apostrophes'
"This is a \"quote\" with quotation marks"
'This string has\nnewlines'
'An em dash: \u2014'
```

#### Raw strings
Prefix a string literal with `r` to make it a raw string. A raw string will not take into account backslash escapes.

```ts
r'Here is a raw string. A newline is represented as \n'
r#'
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

In reality, string-interpolation is syntax sugar. The above desugars to:

```ts
world = 'World';
'Hello, ' + (world)::string + '!'
```

See [String Formatting] for more information.

#### Raw Strings
Strings without encodings are simply represented as byte slices: `uint8[]`. It is simply a string of bytes.

A literal string can be defined as a byte slice by adding `~` before the string:
```ts
~'Null byte: \0' // [78, 117, 108, ..., 58, 32, 0]
```

## Functions
You can abstract a procedure a function which can be called over and over again.

A function may be defined with the `func` keyword:
```ts
func hello_world() {
	println("Hello, world!");
}
```

Then, it may be called by referencing the function (by its name in this case) followed by a call to the function
using parenthesis:
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

The type of functions, and any callable, is written as `(parameters) -> return_type`. For example, the function defined
above has the type `(uint, uint) -> uint`.

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
An anonymous function is a function without a name. They can be declared with the `(parameters) => body` syntax, i.e.:
```ts
let times_two = (val) => 2 * val; // Expression-like
// Or, use a block:
let times_two = (val) => {
	2 * val // Implicit-return
};
println(times_two(5)); // 10
```

All parameters and the return type of anonymous functions can be inferred. All anonymous functions have inferred capturing of outside variables.

### Currying
*⚠️ This section of the specification is still pending and is current an optional part of the Terbium specification.*

Currying is a concept in functional programming that allows partial applications of functions, for example:
```ts
// This is not Terbium
add = (x, y) => x + y
plus_one = add(1)
two = plus_one(1)

// Which could be rewritten as
add = (x) => (y) => x + y
```

Terbium requires that all required arguments are passed when a function is called. A partial application can be done along the lines of:
```ts
func add(x: uint, y: uint) = x + y;
func plus_one(x: uint) = add(1, x);
```

However, that is cumbersome. Therefore, Terbium provides syntax sugar with the `partial` keyword:
```ts
partial func add(x: uint, y: uint) = x + y;

add(5, 2) // 7
add(5)(2) // 7
```

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

The `default` operation is also implemented for many types. Placing a `?` after the parameter name makes the parameter
default to the value returned by the default operation:
```ts
// default op implementation for uint:
extend uint {
	static op func default() = 0;
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

add(1, 1); // 3
```

## Breaking Specification Changes

### 2022 Jun 7 (Pre-release)
Previously, Terbium offered explicit **im**mutability. This change inverses this so that Terbium utilizes explicit mutablility and implicit immutability.

###### Before
```ts
let x = [];
x.push(1);  // x is implicitly mutable so this works

let immut x = [];
x.push(1);  // fails
```

###### Now
```ts
let mut x = [];  // x must be explicitly mutable
x.push(1);

let x = [];
x.push(1);  // fails
```

This change also secures a previously ambiguate decision on whether or not the `let` keyword should be permanent.
Before, this decision was ambiguous between Choices A and B shown below:

###### Choice A
Declaring a variable with `let` gives it an immutable type.

###### Choice B
Declaring a variable with `let` is mandatory if it is not in scope yet.

Choice B is now the standard.
