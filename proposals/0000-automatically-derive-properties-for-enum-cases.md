# Automatically derive properties for enum cases

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Stephen Celis](https://github.com/stephencelis)
* Review Manager: TBD
* Status: **Awaiting implementation**

<!--
*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
-->

## Introduction

Swift should automatically derive properties for enum cases in order to solve two problems:

1. Enums with associated values have been a serious ergonomics issue since Swift's inception. Developers regularly write verbose boilerplate because of this.

2. Key paths are a powerful new Swift feature built around properties. Properties are struct-biased, so enums don't benefit from this feature.

By making the compiler responsible for building enum properties, we unburden developers from the time-consuming task of maintaining and writing partial boilerplate, and we set the stage for enums to benefit from compiler-generated key path code.

Swift-evolution thread: [Automatically derive properties for enum cases](https://forums.swift.org/t/automatically-derive-properties-for-enum-cases/10843/16)

## Motivation

When enums have cases with associated values, they become more difficult to work with. Extracting values requires case analysis, which means a `switch`, `if`, or `guard` statement and block.

``` swift
if case let .value(value) = result {
    // use `value`
}
```

This problem gets worse with closures. Swift privileges single-line closures with both implicit returns _and_ better-inferred return types. Extracting associated values from enums requires multiple lines, which leads to more boilerplate and ceremony.

``` swift
array.filter { result -> Bool in
    if case .value = result {
        return true
    } else {
        return false
    }
}
```

Here we used `if` and `else`. Some teams may prefer to avoid the symmetry of `else` in order to compress things a bit, or to embrace `guard` and enforce an early `return`. All of these statement-based solutions are verbose and read differently. There is no clear winner.

To clean up this boilerplate and confusion, it's common for developers to manually define properties in order to work with expressions, instead.

``` swift
extension Result {
    var value: Value? {
        if case let .value(value) = result {
            return value
        } else {
            return nil
        }
    }
}
```

Such properties can simplify a lot of code! Our earlier example becomes:

``` swift
array.filter { $0.value != nil }
```

This kind of code isn't uncommon. The Swift standard library provides a lot of higher-order methods (`map`, `filter`, `reduce`, `flatMap`, `sorted`, `first(where:)`, etc.) that are easier to use and read when they can be expressed as single lines.

These properties also allow us to traverse deeply into nested structures using optional chaining. _E.g._,

``` swift
result.value?.anotherCase?.name
```

Without these properties, this becomes much more verbose: we have to wade through layers of pattern matching. The most efficient case of nested enums uses nested pattern matching, but still suffers from an additional statement, variable binding, and scope:

``` swift
if case let .value(.anotherCase(anotherCase)) = result {
    // use `anotherCase.name`
}
```

This kind of deep pattern matching isn't commonly known. It's far more common to encounter Swift code that pattern matches over multiple clauses.

``` swift
if
    case let .value(value) = result,
    case let .anotherCase(anotherCase) = value {

        // use `anotherCase.name`  
}
```

This is even more difficult to read.

When types are nested between enums, deep pattern matching isn't even possible: pattern matching _must_ be broken up over multiple clauses.

``` swift
if
    case let .value(myStruct) = result,
    case let .anotherCase(anotherCase) = myStruct.someProperty {

        // use `anotherCase.whatever`
}
// vs.
result.value?.someProperty.anotherCase?.whatever

// or

if
    case let .value(myStruct) = result,
    let firstChild = myStruct.children.first,
    case let .anotherCase(anotherCase) = firstChild {

        // use `anotherCase.whateverStill`
}
// vs.
result.value?.children.first?.anotherCase?.whateverStill
```

Messier still! Optional-chaining reads nicely: left-to-right. It's much more difficult to follow case binding over multiple clauses, and there are more variables to track and reason about.

Currently, all of these properties must be written by hand. Developers waste a lot of time writing and maintaining noisy boilerplate that may only cover a small subsection of the enums defined in their code. The compiler should unburden developers and automate this with generated code.

### Key Paths

The primary motivation explored thus far is ergonomical, but there's another practical reason to derive these properties: doing so will automatically provide key path support for enums.

Key paths are a powerful new Swift language feature that is built on top of properties and is thus biased towards structs.

Given a struct:

``` swift
struct User {
    var name: String
}
```

Swift will automatically generate a key path:

``` swift
\User.name
// WritableKeyPath<User, String>
```

This key path can be used in a number of ways to make code expressive and reduce boilerplate.

Enum cases currently live outside the key path world, but if Swift were to derive properties for enum cases, we'd immediately bridge that gap.

``` swift
enum Result<Value, Other> {
    case value(Value)
    case other(Other)
}

\Result<Int, String>.value
// KeyPath<Result<Int, String>, Int?>
```

## Proposed solution

Swift should automatically derive these properties. Given the following enum:

``` swift
enum Result<Value, Other> {
    case value(Value)
    case other(Other)
}
```

Swift should derive the following properties:

``` swift
extension Result {
    var value: Value? {
        if case let .value(value) = result {
            return value
        } else {
            return nil
        }
    }

    var other: Other? {
        if case let .other(other) = result {
            return other
        } else {
            return nil
        }
    }
}
```

For enum cases with multiple associated values, Swift should derive a property that returns an optional tuple of associated values.

``` swift
enum Color {
    case rgba(red: Float, green: Float, blue: Float, alpha: Float)
    // …
}

// derives:
extension Color {
    var rgba: (red: Float, green: Float, blue: Float, alpha: Float)? {
        if case let .rgba(red, green, blue, alpha) {
            return (red: red, green: green, blue: blue, alpha: alpha)
        } else {
            return nil
        }
    }
    // …
}
```

For enum cases with no associated values, Swift should derive boolean getters.

``` swift
enum TrafficLight {
    case red
    case yellow
    case green
}

// derives:
extension TrafficLight
    var isRed: Bool {
        if case .red = self {
            return true
        } else {
            return false
        }
    }

    var isYellow: Bool { /* … */ }
    var isGreen: Bool { /* … */ }
}

let trafficLight = TrafficLight.red
trafficLight.isRed    // true
trafficLight.isYellow // false
```

In fact, Swift should derive these boolean getters for all enum cases, including those with associated values.

``` swift
extension Result {
    var isValue: Bool { /* … */ }
    var isOther: Bool { /* … */ }
}

extension Color {
    var isRgba: Bool { /* … */ }
    // …
}
```

### Conflicts

This overall solution is in conflict with unimplemented components of an accepted proposal, [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md).

In this proposal, enum cases can be given overlapping case names (as long as the argument labels and types differ). We propose this unimplemented component should be revised to avoid ambiguity and the failure to generate properties.


### Future direction

These getter properties lay the foundation for introducing setters in the future. Mutating a deeply-nested enum requires a lot of code right now. The ability to mutate such values with optional-chaining could improve this immensely.

## Detailed design

N/A

## Source compatibility

This is strictly additive.

Where existing, user-defined properties collide, Swift could not derive properties.

## Effect on ABI stability

N/A

## Effect on API resilience

N/A

## Alternatives considered

### Do not derive new names for boolean getters

There is some resistance to the idea of magically deriving new names for boolean getters.

Enum cases with no associated values could, instead, derive properties that return `Optional<Void>`, which would unify how other properties are generated.

``` swift
enum TrafficLight {
    case red
    case yellow
    case green
}

// derives:
extension TrafficLight
    var red: Void? { /* … */ }
    var yellow: Void? { /* … */ }
    var green: Void? { /* … */ }
}

let trafficLight = TrafficLight.red
trafficLight.red == nil    // false
trafficLight.yellow == nil // true
```

Properties that return `Optional<Void>` are a strange thing to encounter, though, and make the feature more difficult to understand.

Another option for enum cases with no associated values would be to derive boolean getters with the same name as the case.

``` swift
enum TrafficLight {
    case red
    case yellow
    case green
}

// derives:
extension TrafficLight
    var red: Bool { /* … */ }
    var yellow: Bool { /* … */ }
    var green: Bool { /* … */ }
}

let trafficLight = TrafficLight.red
trafficLight.red    // true
trafficLight.yellow // false
```

This reads strangely, though, and seems to go against Swift's API design guidelines.

For these reasons, we prefer to generate boolean properties that are prefixed with `is` where the first letter of the case is made uppercase such that:

- `value` derives `isValue`
- `red` derives `isRed`
- `rgba` derives `isRgba`

This kind of naming seems to work for most cases investigated.

### Do not revise SE-0155

If we choose to implement SE-0155 in full, we need a solution for overlapping property names. Two solutions stand out:

1. Swift could not derive properties for overlapping cases. This consequence is difficult to actively communicate and may be confusing.

2. Swift could support overloaded property names and disambiguate with type hints or labels. Such a change has far greater consequences and is felt to be beyond the scope of this proposal. Such a change could also be made at a later date while favoring the first solution in the interim.

### Generate properties from case labels

It was suggested that "we could cut the compound-payload cloth" and "form accessors from each label" instead. _E.g._,

``` swift
@frozen enum Foo {
    case bar(x: Int, y: String)
    case baz(x: Int, z: Float)
}

let foo: Foo
foo.x // Int, from either bar or baz payload
foo.y // String?, from bar payload
foo.z // Float?, from baz payload
```

This is interesting, but leads to other ambiguities. What if labels overlap?

``` swift
enum Foo {
  case bar(x: Int)
  case baz(x: String)
}

let foo: Foo
foo.x // what type is this?
```

What if labels aren't specified (a common case)?

``` swift
enum Foo {
  case bar(Int)
  case bar(String)
}

let foo: Foo
foo.0 // what type is this?
```

### Require protocol conformance for derivation

Swift could special-case derivation of these properties by requiring a protocol conformance, as in [SE-0167](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md), with `Encodable` and `Decodable`, and as in [SE-0194](https://github.com/apple/swift-evolution/blob/master/proposals/0194-derived-collection-of-enum-cases.md), with `CaseIterable`. This is extra work for the end user, though, which is ideally avoided.

### Use a new syntax instead of properties

A few folks have suggested using a new syntax instead. It was pointed out that enum labels are sometimes written to read like a function. _E.g._,

``` swift
enum Selection {
    case range(from: Int, to: Int)
    case discreteIndices(in: [Int], inverted: Bool)
}
```

There is an argument that property access of such cases pose readability issues.

``` swift
let selection = Selection.range(from: 1, to: 2)
selection.range?.to // = .some(2)
selection.discreteIndices?.in.first
```

This appears to be a minority of cases and, pending implementation of SE-0155, is how tuple binding currently operates in the language.

``` swift
switch selection {
case let range(range):
    range.to // Int
case let discreteIndices(discreteIndices):
    range.in.first // Int?
}
```

Still, some have proposed adding entirely new grammar to the language. For example, a `matches` operator:

``` swift
selection matches .range(from: _, to: let upperBound) // = .some(2)
(selection matches .discreteIndices(let indices, _))?.first

selection matches let .range(a, b) // produces an (a: Int, b: Int)?
```

Or with syntax similar to pattern matching:

``` swift
(case .anotherCase = (case .value = result)?.someOtherProperty)?.name
```

These grammars add weight to the language and lead to a lot of unanswered questions. When bindings are used to express values rather than bind, are those variables made available anywhere in scope? How do such behaviors work with pattern matching as a whole?