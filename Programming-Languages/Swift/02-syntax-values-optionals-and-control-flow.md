---
title: "02 - Syntax, Values, Optionals, and Control Flow"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, syntax, optionals]
aliases: []
---

# 02 - Syntax, Values, Optionals, and Control Flow

[toc]

> **TL;DR:** Swift's surface syntax is compact, but the real foundation is explicit values, strong static types, and optionals. If you learn `let`, `var`, type inference, `if let`, `guard let`, `switch`, and collection literals deeply, most beginner Swift code becomes readable.

## Real-World Example

This example parses a user profile from a dictionary-like payload. It shows constants, variables, optionals, optional binding, `guard`, string interpolation, arrays, and `switch`.

```swift
func describeUser(payload: [String: String]) -> String {
    guard let id = payload["id"], !id.isEmpty else {
        return "Invalid user: missing id"
    }

    let name = payload["name"] ?? "Anonymous"
    let role = payload["role"]

    switch role {
    case "admin":
        return "\(name) (\(id)) can manage the system."
    case "editor":
        return "\(name) (\(id)) can edit content."
    case nil:
        return "\(name) (\(id)) has no assigned role."
    default:
        return "\(name) (\(id)) has a custom role."
    }
}

print(describeUser(payload: ["id": "42", "name": "Ava", "role": "admin"]))
```

Run a single file while you are learning. Later notes move this into a package with tests.

```bash
swift user.swift
```

## Vocabulary

**`let`**: Declares a constant binding. The name cannot be reassigned after initialization.

---

**`var`**: Declares a variable binding. The name can be reassigned if the variable is in a mutable context.

---

**Type inference**: The compiler's ability to infer a type from an initializer or context, such as inferring `String` from `"hello"`.

---

**Optional**: A type that either contains a wrapped value or contains no value. `String?` means "maybe a String."

---

**Optional binding**: `if let`, `guard let`, or `while let` syntax that unwraps an optional only when it contains a value.

---

**Nil-coalescing operator**: The `??` operator. It returns the optional's value when present, or a default when nil.

---

**Exhaustive switch**: Swift requires every possible value to be handled by `switch`, either by concrete cases or `default`.

## Intuition

Swift wants you to make absence explicit. In many languages, a variable of type `String` can secretly be null. In Swift, `String` and `String?` are different types. This is not decoration. It changes what code is legal.

Optionals force the question: "What should happen when this value is missing?" That pressure is useful. Most production bugs around JSON, UI inputs, databases, and network responses start with "we assumed this field was always there."

## Constants and Variables

Use `let` by default. Make a binding `var` only when you need to change it. This is not just style. It gives the compiler and the reader a smaller state space.

```swift
let apiBaseURL = "https://api.example.com"
var retryCount = 0

retryCount += 1
print("Retry \(retryCount) against \(apiBaseURL)")
```

> [!TIP]
> A senior Swift habit is to reduce mutable state before reaching for clever abstractions. If a value does not need to change, make that fact visible with `let`.

## Type Inference and Annotations

Swift inference is local and strong. You usually let the compiler infer obvious types, but annotate public APIs, ambiguous numeric values, and places where the type is part of the design.

```swift
let name = "Nia"                  // String
let maxRetries: Int = 3           // explicit because it is a policy
let timeoutSeconds: Double = 2.5  // explicit to avoid numeric ambiguity
```

## Optionals

An optional is not a value plus a flag you manually check. It is a distinct type with language support. You must unwrap it safely before using the wrapped value as a normal value.

```swift
let rawAge: String? = "34"

if let rawAge, let age = Int(rawAge) {
    print("Age next year: \(age + 1)")
} else {
    print("Age was missing or invalid")
}
```

Use `guard` when missing data means the current function cannot continue. It keeps the successful path unindented.

```swift
func loadProfile(id: String?) -> String {
    guard let id, !id.isEmpty else {
        return "Cannot load a profile without an id."
    }

    return "Loading profile \(id)"
}
```

## Collections

Swift's core collection literals cover arrays, dictionaries, and sets. Arrays and dictionaries are value types with copy-on-write storage, which means passing them around is usually cheap until mutation requires a separate copy.

```swift
let names = ["Ava", "Mina", "Theo"]
let scores = ["Ava": 98, "Mina": 91]
let uniqueNames: Set<String> = ["Ava", "Ava", "Mina"]

print(names.count)
print(scores["Ava"] ?? 0)
print(uniqueNames.contains("Theo"))
```

## Control Flow

Use `if` for boolean branches, `guard` for preconditions, `for` for iteration, and `switch` for domain cases. A Swift `switch` does not fall through by default.

```swift
enum ImportResult {
    case inserted(count: Int)
    case skipped(reason: String)
    case failed(message: String)
}

func report(_ result: ImportResult) -> String {
    switch result {
    case .inserted(let count):
        return "Inserted \(count) rows."
    case .skipped(let reason):
        return "Skipped import: \(reason)."
    case .failed(let message):
        return "Failed import: \(message)."
    }
}
```

## Pitfalls

- **Force unwrapping too early**: `value!` turns a recoverable absence into a crash. Use it only when a crash is the correct invariant failure.
- **Making everything optional**: Optional should model real absence, not uncertainty in your design.
- **Confusing `let` with object immutability**: A `let` reference to a class instance prevents rebinding, not mutation of the object through its mutable properties.
- **Using `default` too quickly in `switch`**: For enums you own, avoid `default` at first. Exhaustive cases let the compiler tell you when the model changes.

## Exercises

1. Write a function that accepts `[String: String]` and returns a validated email string or an error message.
2. Rewrite the function once with nested `if let`, then once with `guard let`. Compare readability.
3. Create an enum for `PaymentState` and handle it with an exhaustive `switch`.
4. Replace a force unwrap in any sample code with safe optional binding.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/controlflow/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/
- Conversation with user on 2026-06-07

## Related

- Previous: [01 - Swift Mastery Map and Current Toolchain](./01-swift-mastery-map-and-current-toolchain.md)
- Next: [03 - Functions, Closures, Error Handling, and Result](./03-functions-closures-error-handling-and-result.md)
- Later: [06 - Memory, ARC, Value Semantics, and Copy-on-Write](./06-memory-arc-value-semantics-and-copy-on-write.md)

