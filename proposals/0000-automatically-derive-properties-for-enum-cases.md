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

This proposal introduces ways to automatically derive enum case properties to enhance user ergonomics and reduce common boilerplate code.

Swift-evolution thread: [Automatically derive properties for enum cases](https://forums.swift.org/t/automatically-derive-properties-for-enum-cases/10843/16)

## Motivation

Automatic property derivation for enum cases solves two problems:

1. **Boilerplate**: Developers regularly write verbose boilerplate to handle associated values. This is a serious ergonomics issue that lingers from Swift's inception.

2. **Key paths**: Enums don't benefit from key paths, a powerful new Swift feature built around struct properties. Enums should participate in these advances.

Allowing the compiler to automatically build enum properties unburdens developers from mainaining and writing boilerplate and sets the stage for enum types to benefit from compiler-generated key path code.

### Associated Values

Enum cases with associated values are more difficult to work with. They require case analysis and pattern matching, complicating usage in `switch`, `if`, or `guard` statements and blocks.

``` swift
if case let .value(value) = result {
    // use `value`
}
```

This challenge increases with closures. Swift privileges single-line closures with both implicit returns and better-inferred return types. It should not take multiple lines to extract associated value information from enum instances. The current technology requires excessive boilerplate and ceremony to access these values.

``` swift
array.filter { result -> Bool in
    if case .value = result {
        return true
    } else {
        return false
    }
}
```

While this example uses `if` and `else`, some teams may prefer to embrace `guard` and enforce and early `return`. Regardless, these statement-based solutions are verbose and there's no single ergonomic best practice for recovering embedded associated values.

Instead, developers commonly hand-define properties to allow associated value cases to work with expressions. For example:

``` swift
extension Result {
    /// Returns associated value as `Optional<Value>`
    var value: Value? {
        if case let .value(value) = result {
            return value
        } else {
            return nil
        }
    }
}
```

Properties reduce code size and enhance readability. Contrast the following property-based enumeration processing with the earlier efforts. Extraction and comparison reduce to a single line.

``` swift
// select items supporting associated value `value`
array.filter { $0.value != nil }
```

This approach is widely applicable. The Swift Standard Library provides several higher-order methods including `map`, `filter`, `reduce`, `flatMap`, `sorted`, `first(where:)`, etc. Each is easier to use, read, and maintain when expressed as single lines.

### Deep Nesting

Enum properties allow deep traversal into nested structures using optional chaining. For example:

``` swift
result.value?.loggedInUser?.name
```

Without properties, filtering by associated value becomes much more verbose, requiring layers of pattern matching. The most efficient approach uses nested pattern matching and suffers from an additional statement, variable binding, and scope. Deep pattern matching is hard to read and unnecessarily obscure:

``` swift
// retrieve `name` from embedded associated value
if case let .value(.loggedInUser(user)) = result {
    // use `user.name`
}
```

Breaking this down into multiple condition clauses remains complex and difficult:

``` swift
// retrieve `name` from embedded associated value
if
    case let .value(value) = result,
    case let .loggedInUser(user) = value {

        // use `user.name`
}
```

When types nest between enum instances, deep pattern matching isn't possible; pattern matching _must_ be broken up over multiple clauses:

``` swift
// retrieve nested enum property from intermediate associated structure value
if
    case let .value(myStruct) = result,
    case let .loggedInUser(user) = myStruct.someProperty {

        // use `user.name`
}

// or

if
    case let .value(myStruct) = result,
    let firstChild = myStruct.children.first,
    case let .anotherCase(theCase) = firstChild {

        // use `theCase.property`
}
```

Compare with the more ergonomic solution produced by this proposal:

``` swift
// access `name`
result.value?.someProperty.loggedInUser?.name

// or

// access `property`
result.value?.children.first?.anotherCase?.property
```

Optional-chaining reads cleanly left-to-right. It eliminates the need to follow case binding through multiple clauses. There are fewer variables to track and reason about.

In the current version of Swift, properties like this must be established by hand by the developer. Developers waste time writing and maintaining noisy boilerplate that covers a small subsection of enums defined in their code. The compiler should unburden developers and automate this access through ergonomic generated code.

### Key Paths

Automatically deriving properties opens the door to key path support for enums. Key paths are a powerful new Swift language feature built on top of properties. As such, they are so-far biased towards structs.

Given a struct:

``` swift
struct User {
    var name: String
}
```

Swift automatically generates a key path:

``` swift
\User.name
// WritableKeyPath<User, String>
```

This key path can be used to enhance code expression and reduce boilerplate.

Enum cases currently live outside the key path world. Swift-derived properties for enum cases would bridge that gap:

``` swift
enum Result<Value, Other> {
    case value(Value)
    case other(Other)
}

\Result<Int, String>.value
// KeyPath<Result<Int, String>, Int?>
```

## Proposed solution

If accepted, Swift will automatically derive enum properties from case names and labels. Given the following enum:

``` swift
enum Result<Value, Other> {
    case value(Value)
    case other(Other)
}
```

Swift derives the following properties:

``` swift
extension Result {
    /// `value` accessor
    var value: Value? {
        if case let .value(value) = result {
            return value
        } else {
            return nil
        }
    }

    /// `other` accessor
    var other: Other? {
        if case let .other(other) = result {
            return other
        } else {
            return nil
        }
    }
}
```

Swift will derive a property returning an optional tuple of associated values for cases with multiple associated values:

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

Enum cases with no associated values use boolean getters.

``` swift
/// Represents possible display states for a traffic light
enum TrafficLight {
    case red
    case yellow
    case green
}

// derives:
extension TrafficLight
    /// automatically-generated boolean getter
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

Swift should derive Boolean getters for all enum cases, including those with associated values.

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

This overall solution is in conflict with unimplemented components of an accepted proposal. [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md) permits enum cases to be given overlapping case names so long as the argument labels and types differ. This unimplemented component must be revised to avoid ambiguity when automatically generating properties.

### Future direction

These getter properties lay the foundation for introducing setters in the future. Mutating a deeply-nested enum currently requires a lot of boilerplate. The ability to mutate such values with optional-chaining would improve this.

## Detailed design

N/A

## Source compatibility

This is strictly additive.

Where existing, user-defined properties collide, Swift cannot derive properties.

## Effect on ABI stability

N/A

## Effect on API resilience

N/A

## Alternatives considered

### Do not derive new names for boolean getters

Magically deriving new names for Boolean getters is not universally popular. Enum cases without associated values could derive properties that return `Optional<Void>`. This unifies property generation across enum types:

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
trafficLight.red == nil // false
trafficLight.yellow == nil // true
```

Properties that return `Optional<Void>` are uncommon and makes this feature more difficult to understand.

Another option for enum cases without associated values is to derive Boolean getters with the same name as the case.

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
trafficLight.red // true
trafficLight.yellow // false
```

This naming is contradicted by Swift's API design guidelines.

For these reasons, we prefer to generate Boolean properties prefixed with `is` where the first letter of the case is made uppercase such that:

- `value` derives `isValue`
- `red` derives `isRed`
- `rgba` derives `isRgba`

### Do not revise SE-0155

If SE-0155 is implemented in full, this proposal requires solution to handle overlapping property names:

1. Swift could exclude automatic property generation for overlapping cases. This consequence is both confusing and difficult to actively communicate.

2. Swift could support overloaded property names, disambiguated with type hints or labels. Such a change has far greater consequences and is felt to be beyond the scope of this proposal. Such a change could also be made at a later date while favoring the first solution in the interim.

### Generate properties from case labels

Feedback from the Swift forums suggested that "we could cut the compound-payload cloth" and "form accessors from each label" instead. _E.g._,

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

While interesting, this leads to further ambiguities. What happens when labels overlap?

``` swift
enum Foo {
  case bar(x: Int)
  case baz(x: String)
}

let foo: Foo
foo.x // what type is this?
```

Similarly, what happens when labels are unspecified (a common case)?

``` swift
enum Foo {
  case bar(Int)
  case bar(String)
}

let foo: Foo
foo.0 // what type is this?
```

### Require protocol conformance for derivation

Swift could special-case property derivation by requiring a protocol conformance, as in [SE-0167](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md), with `Encodable` and `Decodable`, and as in [SE-0194](https://github.com/apple/swift-evolution/blob/master/proposals/0194-derived-collection-of-enum-cases.md), with `CaseIterable`. This adds extra work for the end user, ideally avoided.

### Use a new syntax instead of properties

Feedback from the Swift forums suggested using a new syntax instead. One motivation to avoid properties and adopt a new syntax suggested that enum labels are sometimes written to read like a function. _E.g._,

``` swift
enum Selection {
    case range(from: Int, to: Int)
    case discreteIndices(in: [Int], inverted: Bool)
}
```

It was argued that these cases pose readability issues.

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

A couple new grammars based on pattern matching were suggested. One example:

``` swift
(case .anotherCase = (case .value = result)?.someOtherProperty)?.name
```

Another:

``` swift
selection matches .range(from: _, to: let upperBound) // = .some(2)
(selection matches .discreteIndices(let indices, _))?.first

selection matches let .range(a, b) // produces an (a: Int, b: Int)?
```

These grammars add weight to the language and lead to more unanswered questions. When bindings are used, as in the second example, to express values rather than bind variables, are those variables made available anywhere in scope? How do these solutions work with pattern matching as a whole?

These solutions also fail to provide key path support to enums.
