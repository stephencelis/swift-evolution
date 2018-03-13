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

2. Key paths are a powerful new Swift feature built around properties. Properties are struct-biased, so enums are left behind.

By making the compiler responsible for building enum properties, we unburden developers from the time-consuming task of maintaining and writing partial boilerplate, and we set the stage for enums to benefit from compiler-generated key path code.

Swift-evolution thread: [Automatically derive properties for enum cases](https://forums.swift.org/t/automatically-derive-properties-for-enum-cases/10843/16)

## Motivation

When enums have cases with associated values, they become more difficult to work with. Extracting values requires case analysis, which means a `switch`, `if`, or `guard` statement and block.

``` swift
if case let .value(value) = result {
    // use `value`
}
```

This problem gets worse with closures. Swift privileges single-line closures with implicit returns and better-inferred return types. Extracting associated values requires multiple lines, which leads to more boilerplate and ceremony.

``` swift
array.filter { result -> Bool in
    if case .value = result {
        return true
    } else {
        return false
    }
}
```

Here we used `if` and `else`. Some teams may prefer to avoid the symmetry of `else` to compact things a bit, or to embrace `guard` and enforce an early `return`. All of these statement-based solutions are verbose, though, and read differently. There is no clear winner.

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

These properties also let us traverse deeply into nested structures using optional chaining. Without them, we would have to go through layers of `case let` binding.

``` swift
if
    case let .value(value) = result,
    case let .anotherCase(anotherCase) = value {

        // use `anotherCase.name`  
}

// vs.

result.value?.anotherCase?.name
```

These properties must currently be written by hand. Developers waste a lot of time writing and maintaining noisy boilerplate that may only cover a small subsection of the enums defined in their code.

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

Such a solution is in conflict with an unimplemented component of the accepted proposal, [SE-0155](https://github.com/apple/swift-evolution/blob/master/proposals/0155-normalize-enum-case-representation.md), wherein enum cases can be given the same case name. We propose that this unimplemented component should be revised to avoid ambiguity and the failure to be able to generate properties.

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
trafficLight.red == nil // false
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
trafficLight.red // true
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

### Require protocol conformance for derivation

Swift could special-case derivation of these properties by requiring a protocol conformance, as in [SE-0167](https://github.com/apple/swift-evolution/blob/master/proposals/0167-swift-encoders.md), with `Encodable` and `Decodable`, and as in [SE-0194](https://github.com/apple/swift-evolution/blob/master/proposals/0194-derived-collection-of-enum-cases.md), with `CaseIterable`. This is extra work for the end user, though, which is ideally avoided.