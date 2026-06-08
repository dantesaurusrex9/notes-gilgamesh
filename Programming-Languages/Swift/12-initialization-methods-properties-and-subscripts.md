---
title: "12 - Initialization, Methods, Properties, and Subscripts"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, initialization, properties]
aliases: []
---

# 12 - Initialization, Methods, Properties, and Subscripts

[toc]

> **TL;DR:** Initialization is where Swift forces valid objects into existence. Stored properties, computed properties, property observers, methods, mutating methods, static members, and subscripts are the tools that make types feel native.

## Real-World Example

This example creates a validated `EmailAddress` value. The initializer rejects invalid input, the stored property holds canonical data, and the computed property exposes derived information.

```swift
enum EmailError: Error {
    case invalid
}

struct EmailAddress: Hashable {
    let rawValue: String

    var domain: String {
        rawValue.split(separator: "@").last.map(String.init) ?? ""
    }

    init(_ rawValue: String) throws {
        guard rawValue.contains("@"), rawValue.contains(".") else {
            throw EmailError.invalid
        }
        self.rawValue = rawValue.lowercased()
    }
}

let email = try EmailAddress("Pat@Example.com")
print(email.rawValue)
print(email.domain)
```

## Vocabulary

**Initializer**: Code that creates a valid instance and assigns all stored properties.

---

**Failable initializer**: An initializer that returns nil when construction cannot succeed.

---

**Throwing initializer**: An initializer that throws an error when construction cannot succeed.

---

**Stored property**: A property backed by storage.

---

**Computed property**: A property whose value is computed by code.

---

**Property observer**: `willSet` or `didSet` logic that runs around mutation.

---

**Subscript**: Bracket syntax that lets a type expose indexed or keyed access.

## Intuition

A strong Swift type does not let invalid data leak in and then ask every caller to remember the rules. Put the rule at construction time. If a user ID cannot be empty, reject empty IDs in the initializer. If a percentage must be between 0 and 1, enforce that once.

Initialization is also where class inheritance becomes more complex. Structs and enums are usually simpler: initialize all stored properties and move on. Classes add designated initializers, convenience initializers, superclass initialization, and deinitialization.

## Stored and Computed Properties

Use stored properties for state and computed properties for derived values. Document non-trivial computed property cost.

```swift
struct Rectangle {
    var width: Double
    var height: Double

    var area: Double {
        width * height
    }
}
```

## Property Observers

Use observers for local side effects that belong to the property. Avoid hiding major business workflows inside `didSet`.

```swift
struct Progress {
    var value: Double = 0 {
        didSet {
            value = min(max(value, 0), 1)
        }
    }
}
```

> [!WARNING]
> Property observers can make mutation surprising. Keep them small and obvious, or use explicit methods.

## Methods and Mutating Methods

Struct methods cannot mutate `self` unless marked `mutating`. This makes value mutation visible in the type's API.

```swift
struct RetryPolicy {
    private(set) var attempts = 0

    mutating func recordFailure() {
        attempts += 1
    }
}
```

## Static Members

Use static members for shared constants, factories, and type-level behavior.

```swift
struct Timeout {
    let seconds: Double

    static let defaultRequest = Timeout(seconds: 10)
}
```

## Subscripts

Subscripts make sense when bracket access is natural for the domain: collections, grids, matrices, caches, and keyed stores.

```swift
struct Grid<Element> {
    let width: Int
    private var storage: [Element]

    init(width: Int, storage: [Element]) {
        self.width = width
        self.storage = storage
    }

    subscript(row: Int, column: Int) -> Element {
        get { storage[row * width + column] }
        set { storage[row * width + column] = newValue }
    }
}
```

## Pitfalls

- **Invalid states after init**: If every caller has to remember validation, the type is too weak.
- **Heavy computed properties**: Callers assume property access is cheap unless told otherwise.
- **Surprising `didSet` side effects**: Use explicit methods for meaningful workflows.
- **Subscript for non-indexed work**: Do not hide network calls or expensive database queries behind brackets.
- **Class initializer complexity**: Prefer composition and structs unless class identity is required.

## Exercises

1. Build a `NonEmptyString` type with a throwing initializer.
2. Add a computed property and document its complexity.
3. Add a `mutating` method to a struct and explain why the keyword is required.
4. Implement a two-dimensional grid subscript with bounds checks.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/methods/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/subscripts/
- Conversation with user on 2026-06-07

## Related

- Previous: [11 - Standard Library, Strings, Collections, and Algorithms](./11-standard-library-strings-collections-and-algorithms.md)
- Next: [13 - Access Control, Modules, Packages, and DocC](./13-access-control-modules-packages-and-docc.md)
- Earlier: [04 - Types: Structs, Classes, Enums, and Protocols](./04-types-structs-classes-enums-and-protocols.md)

