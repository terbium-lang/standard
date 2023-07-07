# The `else` operator

*Optionals* are a feature in Terbium which allows the handling of values which may not exist. An anti-feature of Terbium is the lack of a *null*, *nil*, or *undefined* value like other languages.

Instead, Terbium has *optional values*, which have the *optional type* `T?` where `T` is the type of the value that could or could not exist. Optional values in Terbium must be *explicitly handled* to ensure "none" or "absent" values are properly handled and accounted for before runtime.

As a developing language, Terbium's syntax features with optionals are currently unsettled. This proposal which introduces the `else` operator is a potential syntax feature that can be used to easily handle "none" values.

## Proposal

This proposal will introduce the `else` keyword as a *binary infix operator* over any two expressions, where the left operand is of type `T?`, where `T` can be of any type:

```bnf
<else> ::= <expr> "else" <expr>
``` 

### Rule 1

Given `a else b` where `a: T?` and `b: T`, where `T` can be any type:

* if `a` "exists" or is a "some" value, return the inner value of `a` as just `T`.
* if `a` is "absent" or is the "none" value, evaluate and return `b`.

In other languages, this could act similar to *"null or"* or *"unwrap or"*.

*TL;DR: `T? else T -> T`*

### Rule 2

Given `a else b` where `a: T?` and `b: T?`, where `T` can be any type:

* if `a` "exists" or is a "some" value, return the value of `a` still wrapped as a "some" value, still as a `T?`.
* if `a` is "absent" or is the "none" value, evaluate and return the optional value `b`.

In other languages, this could be said to be *"null or to another nullable value"* or in Rust, specifically, the `Option::or_else` method.

*TL;DR: `T? else T? -> T?`*

### Example

Assuming `func some<T>(x: T) -> T? = x;` and `none` is the "absent" value:

```swift
// Rule 1: some(a) else b -> a
let x: int32? = some(5);
let y: int32 = x else 10; // some(5) else 10 = 5
assert_eq(y, 5);

// Rule 1: none else b -> b
let x: int32? = none;
let y: int32 = x else 10; // none else 10 = 10
assert_eq(y, 10);

// Rule 2: some(a) else (some(b) | none) -> some(a)
let x: int32? = some(5);
let y: int32? = x else none; // some(5) else none = some(5)
assert_eq(y, some(5));

// Rule 3: none else (some(b) | none) -> (some(b) | none)
let x: int32? = none;
let y: int32? = x else none; // none else none = none
assert_eq(y, none);
```

## Getting rid of `if-else`

Making `else` an operator all together actually makes a lot of sense. If an if-expression returned an optional value, where
the output is `some(result)` if the true-block executed and `none` if not, then the `else` operator could simply operate on that optional value to provide a fallback block:

```swift
if x < 10 {
    println("x is less than 10");
} /* void? */ else {
    println("x is at least 10");
} // void? else void -> void
```

Similarly, the `T? else T? -> T?` rule was designed to work well with `if-else if`:

```swift
if x < 10 {
    println("x is less than 10");
} /* void? */ else /* begin another if-expression */ if x < 20 {
    println("x is less than 20");
} /* void? */
// void? else void? -> void?
```

...which can then be terminated with an `else`:

```swift
if ... else if ... /* void? */ else {
    println("x is at least 20");
}  // void? else void -> void
```

This in turn allows you to do "partial if statements" like this:

```swift
let x = 5;
let partial /* : void? */ = if x == 5 {
    println("x is 5");
};

// later...
partial else {
    println("x isn't 5");
}
```

...which just makes sense--if you replaced *partial* with the initial if-expression, it would recreate a natural `if-else` statement.

*Note: blocks would need to be handled separately from standard expressions in `else ...` to omit the semicolon*

### The if-then-else ternary operator

The if-then-else ternary operator could be reduced to just an if-then binary operator:

```swift
let x /* : int? */ = if cond then 5;
let y /* : int  */ = x else 10;
let z /* : int? */ = x else if cond2 then 10;
```

When combined with `else`, precedence will be a bit funky:

```swift
let x = if cond then 5 else 10;
// Should be the equivalent of:
let x = (if cond then 5) else 10;
// and not:
let x = if cond then (5 else 10);
```

## Getting rid of all `?-else`

Why handle any `else` as a special-case? For any construct, such as breakable loops,
where `else` was previously accepted could be changed to evaluate into an optional value:

```swift
// if the while loop breaks with a value, result in some(value), otherwise result in none
let result: int32? = while x < 10 { ... };
// then
result else {
    println("loop did not break");
}
```
