# Struct Constructor Proposal Syntax

Note: Structs may only have **one** constructor, called the *primary constructor*. This constructor is exposed as the struct's name as a *function pointer* to the parent scope.
There may be associated methods on the struct which return some form of `Self` that internally construct using the primary constructor (for example, `T.default()` or `T.new()`).

This proposal assumes:

```ts
struct Point {
    x: int,
    y: int,
}
```

## Default constructor

All structs create a default constructor by default if one is not defined:

```ts
// Constructor with the default constructor
let point = Point(x: 0, y: 0); // Point: (*, x: int, y: int) -> Self
```
...where all arguments are strictly keyword arguments. If default values are given:

```ts
struct Point {
    x: int,
    y: int = 0,
}
```
...then they will be optional in the default constructor:
```ts
let point = Point(x: 0); // Point: (*, x: int, y: int = 0) -> Self
```

## Custom constructor

You may define a *custom primary constructor* by adding an argument list after the struct name:

```ts
struct Point(x: int, y: int) {
    x: int = x,
    y: int = y,
}
```

In this case, **all fields must be provided with values** through the use of `=` (see above).

We can then construct our struct like so:
```ts
let point = Point(0, 0); // We don't have to use keyword arguments since we didn't define it as so
```

## Constructor visibility

By default, the constructor is the most restrictive setter visibility out of the struct's fields. For example:

```ts
mod m {
    public struct Point {
        public(get, mod set) x: int,
        public y: int,
    }
}

let point = m.Point(x: 0, y: 0); // error! visibility is limited to `mod`
```

You can modify the visibility of the constructor using the `@constructor(<vis>)` decorator:
```ts
mod m {
    @constructor(public)
    public struct Point {
        public(get, mod set) x: int,
        public y: int,
    }
}

let point = m.Point(x: 0, y: 0); // no error
```

The same applies with a custom constructor:
```ts
mod m {
    @constructor(public)
    public struct Point(v: int) {
        public(get, mod set) x: int = v,
        public y: int = v,
    }
}

let point = m.Point(10); // no error
```

## Post-init

You may run something after the constructor is called with an `op func construct`:

```ts
struct Point {
    x: int,
    y: int,

    op func construct(self) {
        println($"New point at {(self.x, self.y)}");
    }
}

let a = Point(x: 5, y: 10);
let b = Point(x: 3, y: 6);

// New point at (5, 10)
// New point at (3, 6)
```

## Associated/secondary constructors

Associated methods that return a form of `Self` will create the instance using the primary constructor (as it is the only way to create a new instance at its core):

```ts
struct Point(x: int, y: int) {
    x: int = x,
    y: int = y,

    func origin() = Self(0, 0);
    func from_polar(r: float, theta: float) = Self(r * theta.cos(), r * theta.sin());

    op func add(self, other: Self) = Self(self.x + other.x, self.y + other.y);
}
```

## Custom constructor shorthand

By adding a visibility modifier to an argument in a custom constructor, it implies that it will also be a field on that class with the same type and visibility:

```ts
struct Point(public x: int, public y: int) {}

let p = Point(0, 0);
```
