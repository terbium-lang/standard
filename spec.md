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
func main() {}
```
```ts
const main = () => {}
```
```ts
class main {
    op construct() -> void {}
}
```

The following will **not** resolve:
```ts
const main = 5 // 5 is not callable
```
```ts
let main;

func resolve_main() {
    main = () => {}
}
```
```ts
#{
    func main() {}
}
```

### Optional Parameters
`main` can optionally take a single parameter, `args` it is a `string[]` of the arguments passed to the `terbium` command.

```ts
// terbium main.trb arg
require std;

func main(args) {
    std.println(args)  // ['main.trb', 'arg']
}
```

## Strings
A string should be stored as a raw `bytestring` with a specified encoding. A `string` cannot be left without an encoding.
By default, all strings will be in `utf-8` encoding.

### String Literals
A string can be defined by surrounding the contents of the string with either `'` or `"`.
Multiline strings can be formed by surrounding the contents of the string with `#"` and its mirror counterpart (`"#`).

By default, string literals will be encoded in `utf-8`.

```ts
'Single quoted string'
"Double quoted string"

#"Multi-line
string
"#
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

[1] Because strings will **always** have an encoding, null-bytes can only be placed in `bytestring`s.
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
$'Hello, {world}!'  // Hello, World!
```

See [String Formatting] for more information.

#### Literal bytestrings
Strings without encodings can be represented in their lower-level format, `bytestring`. It is simply a string of bytes.

A `bytestring` can be defined literally by adding `~` before the string:
```ts
~'Null byte: \0'
```

