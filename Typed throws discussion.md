# Typed throws discussion

## General

Options we have developing the language while keeping compatibility: https://forums.swift.org/t/typed-throws/39660/66

So it seems there is no law that forbids *thinking* about small source adjustments. ;)

## `throws T` syntax ambiguity

### Issue

> we discovered a design flaw in the current syntax
> if we make the syntax like so:

```swift
func foo() throws SomeError {}
```

> then this is ambiguous:

```swift
protocol Foo {
    func bar1() throws
    mutating
    func bar2()
}
```

> `mutating` is one of several contextual keywords, which can also be an identifier.

> in the protocol `Foo` we could either have this:

```swift
func foo1() throws mutating // () throws mutating -> ()
func foo2() // () -> ()
```

> or that:

```swift
func foo1() throws // () throws Error -> ()
mutating func foo2() // mutating () -> ()
```

> so its ambiguous

### Solution 1: `throws (T)`

current implementation

### Solution 2: `throws<T>`

## `rethrows`

### Issue

> the key question with rethrows, which is whether the signature of A below is equivalent to that of B or that of C

```swift
// A
func foo(fn: () throws -> Void) rethrows

// B
func foo<erased E: Error>(fn: () throws E -> Void) rethrows E

// C
func foo<erased E: Error>(fn: () throws E -> Void) rethrows Error
```

see https://forums.swift.org/t/typed-throws/39660/53

### Solution 1: `rethrows T`

rethrow without converting (compatible to current)

```swift
func foo(_ bar: () throws /* Error */ -> ()) rethrows /* Error */ {
    try bar()
}
```

rethrow with converting (compatible to current)

```swift
func foo(_ bar: () throws /* Error */ -> ()) rethrows /* Error */ {
    do {
        try bar()
    } catch {
        throw CustomError(base: error)
    }
}
```

rethrow without converting

```swift
func foo<E>(_ bar: () throws E -> ()) rethrows E {
    try bar()
}

// If every function implicitly threw Never, we could write
func foo<E>(_ bar: () throws E -> ()) throws E {
    try bar()
}
// which would semantically be the same
```

rethrow with converting

```swift
func foo<E>(_ bar: () throws E -> ()) rethrows CustomError {
    do {
        try bar()
    } catch {
        throw CustomError(base: error)
    }
}
```

see https://forums.swift.org/t/typed-throws/39660/73

Example combinations:

```
func foo(closure: () throws -> ()) rethrows
func foo(closure: () throws -> ()) rethrows IntError
func foo(closure: () throws SomeError -> ()) rethrows
func foo(closure: () throws SomeError -> ()) rethrows IntError
func foo<E: Error>(closure: () throws E -> ()) rethrows E
```

In general:

- `rethrows T`: Throw `T` if parameter throws
- `rethrows === (rethrows T where T === Error)`: Throw `Error` if parameter throws

### Solution 2: `rethrows` always throws input errors or erases to `Error`

> I'd like to strongly encourage a direction that does /not/ worry about or consider this, particularly with a goal to simplify the implementation and surface area complexity of the feature. A context like the above could/should just immediately type erase to `Error` . In addition to being simple, general, and predictable, this is important for compatibility with existing logic.

see https://forums.swift.org/t/typed-throws/39660/152

open questions see https://forums.swift.org/t/typed-throws/39660/160

## `throw` and type inference

### Issue

```swift
struct Foo: Error { ... }
struct Bar: Error { ... }
var throwers = [{ throw Foo() }] // Inferred as `Array<() throws -> ()>`, or `Array<() throws Foo -> ()>`?
throwers.append({ throw Bar() }) // Compiles today, error if we infer `throws Foo`
```

see https://forums.swift.org/t/typed-throws/39660/70

### Solution group: Modify type inference

#### Solution #5

> `throws` functions should never be inferred to be the typed throw but the base `throws` , unless there is an explicitly specified typed throws. It would behave like follows

see https://forums.swift.org/t/typed-throws/39660/175

#### (incomplete) Solution #1

> So don't infer `throws Foo` , and infer `throws Error` instead. This will preserve source compatibility.

see https://forums.swift.org/t/typed-throws/39660/71

**New Issue:**

```swift
func foo() throws SomeError {
  throw SomeError() // can this compile if we don't infer SomeError but Error?
}
```

#### (incomplete) Solution #2

> `throws` functions should never be inferred to be the typed throw but the base `throws` in collections (array, set), unless the type is specified as follows:

see https://forums.swift.org/t/typed-throws/39660/72

**New Issue:**

```swift
struct Foo: Error { ... }
struct Bar: Error { ... }
let closure = [{ throw Foo() }] // `closure` gets inferred to `() throws Foo -> ()`
var throwers = closure // `throwers` gets inferred to `[() throws Foo -> ()]`
throwers.append({ throw Bar() }) // error: type mismatch
```

### Solution group: Attribute `throws`

#### Solution #6

```swift
@typed throw SomeError()
```

This will always use the most specific type inference.

### Solution group: Break source compatibility

#### Solution #4

> I guess the cleanest non source compat solution would be, if we could update source in Swift 6 from `throw FooError()` to `throw FooError() as Error` for all `throw` that happen directly in a `do` block or in closure that has it's type inferred.

see https://forums.swift.org/t/typed-throws/39660/123?

#### (incomplete) Solution #3

```swift
func foo<T>(_: () throws T -> ()) { ... }
let _ = { throw Foo.error } // inferred as '() throws -> ()'
foo({ throws Foo.error }) // inferred as '() throws Foo -> ()'
```

> It also seems fairly likely to me that the source break wouldn't be that large (unless there's a more common use-case I haven't thought of?). It might make sense to tighten up the inference behavior as a change in Swift 6, where the Swift 5 compatibility mode would continue to infer the throws type as Error.

see https://forums.swift.org/t/typed-throws/39660/77

**New Issue:**

```swift
foo({ throws Foo.error }) // inferred as '() throws Foo -> ()'
```

Produces a conflict with generic rethrowing functions that should be introduced to the standard library (like a generic map).
