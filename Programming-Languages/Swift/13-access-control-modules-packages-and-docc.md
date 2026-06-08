---
title: "13 - Access Control, Modules, Packages, and DocC"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, modules, docc]
aliases: []
---

# 13 - Access Control, Modules, Packages, and DocC

[toc]

> **TL;DR:** Swift codebases scale through module boundaries. Access control, package targets, public API design, semantic versioning, and DocC documentation decide whether a library remains maintainable as it grows.

## Real-World Example

This example exposes a small public API while keeping implementation details internal. Callers can create and use `RateLimiter`, but cannot mutate its storage directly.

```swift
public struct RateLimiter {
    private let maxRequests: Int
    private var count: Int = 0

    public init(maxRequests: Int) {
        self.maxRequests = maxRequests
    }

    public mutating func allowRequest() -> Bool {
        guard count < maxRequests else {
            return false
        }

        count += 1
        return true
    }
}
```

## Vocabulary

**Module**: A separately compiled unit imported with `import`.

---

**Target**: A SwiftPM or Xcode build unit that produces a module or bundle.

---

**Access control**: Visibility rules such as `private`, `fileprivate`, `internal`, `package`, `public`, and `open`.

---

**Public API**: Declarations visible to clients outside the module.

---

**ABI**: Application Binary Interface. The binary-level contract between compiled components.

---

**DocC**: Apple's documentation compiler for Swift symbols, articles, and tutorials.

## Intuition

A module is a wall. Inside the wall, you can refactor aggressively. Across the wall, every public declaration becomes a promise. Senior Swift developers design module boundaries so that dependencies flow in one direction and implementation details stay hidden.

Access control is not bureaucracy. It is how you stop accidental coupling. If a symbol does not need to be public, keep it internal or private. Future you will have more freedom.

## Access Levels

Use the narrowest access that supports the intended use.

| Level | Use |
| :--- | :--- |
| `private` | Only within the enclosing declaration or extension scope |
| `fileprivate` | Anywhere in the same file |
| `internal` | Anywhere in the same module, default level |
| `package` | Anywhere in the same package |
| `public` | Visible to external modules, subclassing/overriding limited |
| `open` | Public and subclassable/overridable outside the module |

## Package Boundaries

A useful package layout separates domain, infrastructure, executable, and tests.

```text
Package.swift
Sources/
  Core/
  HTTPClient/
  AppExecutable/
Tests/
  CoreTests/
  HTTPClientTests/
```

The direction should be boring: app depends on features, features depend on core, core depends on almost nothing.

## Documentation Comments

Doc comments are part of API design. A good summary explains what a declaration does and what it returns without restating the obvious.

```swift
/// Allows a fixed number of requests before rejecting additional work.
public struct RateLimiter {
    /// Returns `true` when the request is allowed.
    ///
    /// Complexity: O(1).
    public mutating func allowRequest() -> Bool {
        true
    }
}
```

## DocC Workflow

DocC can generate structured documentation for public Swift symbols and article pages. For packages, use the SwiftPM DocC plugin or Xcode's Build Documentation command.

```bash
swift package generate-documentation
```

For libraries, document:

- What the type does.
- What invariants callers must satisfy.
- Complexity of non-trivial operations.
- Thread-safety or actor-isolation expectations.
- Error cases and cancellation behavior.

## Versioning

Public packages should follow semantic versioning. Breaking public API changes require a major version bump. Adding a new public method is usually minor. Fixing a bug without API change is patch.

```text
1.2.3
major.minor.patch
```

## Pitfalls

- **Making everything public**: Public API is expensive to change.
- **Open classes by default**: `open` is a commitment to subclassing behavior. Avoid it unless you are designing inheritance.
- **Leaky modules**: If every target imports every other target, boundaries are fake.
- **Undocumented concurrency guarantees**: Say whether APIs are thread-safe, actor-isolated, or caller-isolated.
- **Docs that repeat names**: "Gets user" is not useful. Explain behavior, constraints, and failure.

## Exercises

1. Split a toy package into `Core`, `CLI`, and `CoreTests`.
2. Change one public type to internal and explain what client code can no longer do.
3. Write DocC comments for a throwing async method.
4. Identify one type that should be `public` and three helpers that should stay internal.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/
- https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/
- https://www.swift.org/documentation/docc/distributing-documentation-to-other-developers
- https://www.swift.org/documentation/api-design-guidelines/
- Conversation with user on 2026-06-07

## Related

- Previous: [12 - Initialization, Methods, Properties, and Subscripts](./12-initialization-methods-properties-and-subscripts.md)
- Next: [14 - Macros, Property Wrappers, and Metaprogramming](./14-macros-property-wrappers-and-metaprogramming.md)
- Earlier: [08 - SwiftPM, Testing, Compiling, and Shipping](./08-swiftpm-testing-compiling-and-shipping.md)

