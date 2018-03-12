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

I propose that Swift should automatically derive properties for enum cases. It's common to manually define properties for the sake of ergonomics: _e.g._, preferring expressions to `case let` statements. The compiler should generate this code for us. Adding these properties universally will unlock the ability to traverse into deeply nested structures using optional chaining, as well as begin to give enums more equal footing to structs in the key path world.

Swift-evolution thread: [Automatically derive properties for enum cases](https://forums.swift.org/t/automatically-derive-properties-for-enum-cases/10843/16)

## Motivation

When enums have cases with associated values, they become more difficult to work with. Extracting values requires case analysis, which means a `switch`, `if`, or `guard` block.

``` swift
if case let .value(value) = result {
    // use `value`
}
```

This problem can be amplified when working with closures. Swift privileges single-line closures with implicit returns and better-inferred return types. This can lead to a lot more boilerplate and ceremony.

``` swift
array.filter { result -> Bool in
    if case .value = result {
        return true
    } else {
        return false
    }
}
```

Here we used `if` and `else`, though some teams may prefer to avoid `else` to compact things a bit, or to embrace `guard` to enforce the early `return`. All of these statement-based solutions are verbose and read differently. There is no clear winner.

To clean up this boilerplate and confusion, it's common to define properties and work with expressions, instead.

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

These properties can simplify a lot of code! Our earlier example becomes:

``` swift
array.filter { $0.value != nil }
```

These properties also let us traverse deeply into nested structures with optional chaining. Without them, we have to go through layers of `case let` binding.

``` swift
if
    case let .value(value) = result,
    case let .anotherCase(anotherCase) = value {

        // use `anotherCase.name`  
}

// vs.

result.value?.anotherCase?.name
```

Building these properties is a manual process, though, so engineers end up wasting a lot of time writing a lot of noisy boilerplate by hand that may only cover a small subsection of the enums defined in their code.

<!--
Describe the problems that this proposal seeks to address. If the
problem is that some common pattern is currently hard to express, show
how one can currently get a similar effect and describe its
drawbacks. If it's completely new functionality that cannot be
emulated, motivate why this new functionality would help Swift
developers create better Swift code.
-->

## Proposed solution

I propose that Swift automatically derive these properties. For example, given the following enum:

``` swift
enum Result<Value, Other> {
    case value(Value)
    case other(Other)
}
```

Swift would derive the following properties:

``` swift
extension Result {
    var value: Value? {
        if case let .value(value) = result {
            return value
        } else {
            return nil
        }
    }

    var other: Other? { /* … */ }
}
```

For enum cases with multiple associated values, Swift will derive a property that returns an optional tuple of associated values.

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

For enum cases with no associated values, Swift could derive boolean getters.

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

In fact, Swift could derive these boolean getters for all enum cases, including those with associated values.

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

This solution is in conflict with an unimplemented component of [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md), where enum cases may be given the same case name. We propose that this unimplemented component is revised to avoid ambiguity or the failure to generate properties.

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

### Do not generate boolean getters

There is some resistance to the idea of magically deriving new names for boolean getters. Enum cases without associated values could generate boolean getters with the same name as the case, instead.

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

This may read strangely, though, and seems to go against Swift's API design guidelines.

Another option would be to derive properties that return `Optional<Void>`, instead, which would unify how all properties are generated.

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

Properties that return `Optional<Void>` may be a strange thing to encounter, though.

For these reasons, we prefer to generate boolean properties that are prefixed with `is` and where the first letter of the case is made uppercase, such that:

- `value` derives `isValue`
- `red` derives `isRed`
- `rgba` derives `isRgba`

This kind of naming seems to work for most cases investigated.

### Do not revise SE-0155

For enums with case names that overlap:

1. Swift could not derive properties for overlapping cases. This consequence isn't actively communicated, though, and may be surprising.

2. Swift could allow for overloaded property names that could be disambiguated with explicit types. Such a change has far greater consequences and is beyond the scope of this proposal. Such a change could also be made at a later date.

### Require protocol conformance for derivation

Swift could only derive these properties for `enum`s that conform to a protocol, as in [SE-0167](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md) with `Encodable` and `Decodable`, and as in [SE-0194](https://github.com/apple/swift-evolution/blob/master/proposals/0194-derived-collection-of-enum-cases.md), with `CaseIterable`. It would be ideal to avoid pushing this extra work onto the user.