# Key path getter promotion

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Stephen Celis](https://github.com/stephencelis), [Brandon Williams](https://www.github.com/mbrandonw)
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

This proposal introduces a Swift feature that promotes key paths to getter functions wherever appropriate.

Swift-evolution thread: [TODO](TODO)

## Motivation

It's common to write one-off closures that merely traverse from a root type into a value. For example, take the following `User` struct:

``` swift
struct User {
    let email: String
    let isAdmin: Bool
}
```

Given an array of users, we can use `map` to gather an array of their emails.

``` swift
users.map { $0.email }
```

We can also filter our array of users on `isAdmin` to gather an array of admins.

``` swift
users.filter { $0.isAdmin }
```

These ad hoc closures are short and sweet, but Swift already generates this code in the form of key paths.

The forum has [previously proposed](https://forums.swift.org/t/pitch-support-for-map-and-flatmap-with-smart-key-paths/6073) adding `map` and `flatMap` overloads that accept key paths as input. Popular libraries have [added key path overloads](https://github.com/ReactiveCocoa/ReactiveSwift/search?utf8=✓&q=KeyPath&type=), as well.

Adding an overload per function is a losing battle, though. It would be preferable to have first-class support from the language.

## Proposed solution

Swift should allow key paths of the form `KeyPath<Root, Value>` wherever functions of the form `(Root) -> Value` are allowed.

Our earlier examples end up becoming a little bit shorter.

``` swift
users.map(\.email)

users.filter(\.isAdmin)
```

### Prior art

This feature would fall in the footsteps of Ruby, which allows `Symbol#to_proc` to do a similar traversal.

``` ruby
users.map(&:email)

users.filter(&:isAdmin)
```

## Detailed design

N/A

## Source compatibility

This is strictly additive.

## Effect on ABI stability

N/A

## Effect on API resilience

N/A

## Alternatives considered

### Prefix operator

If we do not want compiler magic doing the automatic promotion, we could introduce a prefix operator, `^` for example, to the Standard Library. We can define it today:

``` swift
prefix operator ^

prefix func ^ <Root, Value>(keyPath: KeyPath<Root, Value>) -> (Root) -> Value {
  return { root in root[keyPath: keyPath] }
}
```
