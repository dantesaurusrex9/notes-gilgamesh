---
title: "05 - Generics, Existentials, and API Design"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, generics, api-design]
aliases: []
---

# 05 - Generics, Existentials, and API Design

[toc]

> **TL;DR:** Generics let you express reusable code without throwing away type information. Existentials let you store "any value conforming to a protocol." Senior Swift design means knowing when to use concrete types, `some`, `any`, associated types, and clear names.

## Real-World Example

This example builds a typed repository interface. The generic version preserves the exact model type, while the existential array stores mixed values that share a protocol.

```swift
protocol Identified {
    associatedtype ID: Hashable
    var id: ID { get }
}

struct User: Identified {
    let id: String
    let name: String
}

struct Repository<Model: Identified> {
    private var models: [Model.ID: Model] = [:]

    mutating func insert(_ model: Model) {
        models[model.id] = model
    }

    func find(id: Model.ID) -> Model? {
        models[id]
    }
}

var users = Repository<User>()
users.insert(User(id: "u-1", name: "Ava"))
print(users.find(id: "u-1")?.name ?? "missing")
```

## Vocabulary

**Generic type**: A type parameterized over another type, such as `Array<Element>` or `Repository<Model>`.

---

**Type parameter**: A placeholder type name inside a generic declaration, such as `Element`, `T`, or `Model`.

---

**Constraint**: A requirement on a generic parameter, such as `Model: Identified`.

---

**Associated type**: A placeholder type declared by a protocol and filled in by each conforming type.

---

**Opaque type**: A `some Protocol` return or parameter that hides the concrete type from callers while preserving one concrete type chosen by the implementation.

---

**Existential type**: An `any Protocol` value that can hold any concrete value conforming to that protocol.

## Intuition

Generics are for preserving information. If a function accepts `[User]` and returns `[User]`, the compiler knows the exact element type. If it accepts `[any Identified]`, the compiler knows only the protocol contract at the use site. Both are valid, but they solve different problems.

Use this decision rule:

| Need | Prefer |
| :--- | :--- |
| One concrete type flows through the API | Concrete type |
| Reusable algorithm over many types | Generic parameter |
| Hide the exact return type but keep static dispatch | `some Protocol` |
| Store mixed conforming values | `any Protocol` |
| Framework/plugin boundary | Existential or type-erased wrapper |

## Generic Functions

Start with a generic function when the implementation is identical across types and the type relationship matters.

```swift
func firstDuplicate<Element: Hashable>(in values: [Element]) -> Element? {
    var seen = Set<Element>()

    for value in values {
        if seen.contains(value) {
            return value
        }
        seen.insert(value)
    }

    return nil
}

print(firstDuplicate(in: [1, 2, 3, 2]) ?? -1)
```

## Associated Types

Associated types make protocols more expressive, but they also make plain existential storage more complicated. Use them when the relationship is essential. This parser example defines its own error so it can run independently from the previous note.

```swift
enum ParseError: Error {
    case notANumber(String)
}

protocol Parser {
    associatedtype Output
    func parse(_ input: String) throws -> Output
}

struct IntParser: Parser {
    func parse(_ input: String) throws -> Int {
        guard let value = Int(input) else {
            throw ParseError.notANumber(input)
        }
        return value
    }
}
```

## `some` Versus `any`

`some` means "there is one concrete type here, but I am not naming it." `any` means "this value can be any conforming type, and dynamic dispatch may be involved."

```swift
protocol Renderer {
    func render() -> String
}

struct TextRenderer: Renderer {
    let text: String
    func render() -> String { text }
}

func makeRenderer() -> some Renderer {
    TextRenderer(text: "Hello")
}

let renderer: any Renderer = TextRenderer(text: "Stored as an existential")
print(makeRenderer().render())
print(renderer.render())
```

## API Design

Swift's official API guidelines are blunt: clarity at the point of use is the most important goal, and clarity is more important than brevity. That should shape your names, labels, return types, and overloads.

```swift
// Clear call site:
cache.insert(imageData, for: imageURL)

// Worse:
cache.put(imageData, imageURL)
```

Document complexity when it is not obvious. A computed property that scans a collection should not look like a cheap stored property.

```swift
extension Array where Element == Int {
    /// Returns the largest value by scanning the whole array.
    ///
    /// Complexity: O(n).
    var largestValue: Int? {
        self.max()
    }
}
```

## Pitfalls

- **Generic code without a payoff**: If only one concrete type exists and no algorithm is shared, a generic may be noise.
- **`any` where `some` would preserve type information**: Existentials are useful, but they erase information and can affect dispatch and optimization.
- **Protocols that mirror one concrete type**: A protocol with exactly one conformer may be premature unless it exists for testing or module boundaries.
- **Clever short names**: Swift call sites should read clearly. A short label that requires memory is worse than a longer label that explains intent.
- **Leaking implementation types in public APIs**: For libraries, choose public return types deliberately because changing them is a compatibility event.

## Exercises

1. Write a generic `Queue<Element>` with `enqueue` and `dequeue`.
2. Create a `Parser` protocol with an associated `Output`, then implement parsers for `Int` and `URL`.
3. Write one function returning `some Renderer` and one property storing `any Renderer`. Explain the difference.
4. Rename a vague method so its call site reads like a sentence.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/
- https://www.swift.org/documentation/api-design-guidelines/
- https://download.swift.org/docs/assets/generics.pdf
- Conversation with user on 2026-06-07

## Related

- Previous: [04 - Types: Structs, Classes, Enums, and Protocols](./04-types-structs-classes-enums-and-protocols.md)
- Next: [06 - Memory, ARC, Value Semantics, and Copy-on-Write](./06-memory-arc-value-semantics-and-copy-on-write.md)
- Later: [10 - Senior-Level Swift Engineering Habits](./10-senior-level-swift-engineering-habits.md)
