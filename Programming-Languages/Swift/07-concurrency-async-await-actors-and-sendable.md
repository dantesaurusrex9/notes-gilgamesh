---
title: "07 - Concurrency: Async, Await, Actors, and Sendable"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, concurrency, actors]
aliases: []
---

# 07 - Concurrency: Async, Await, Actors, and Sendable

[toc]

> **TL;DR:** Swift concurrency is not just nicer callback syntax. It is a type-system-backed model for async work, structured tasks, actor isolation, and data-race safety. In Swift 6 language mode, many unsafe cross-concurrency patterns become compiler errors.

## Real-World Example

This example uses an actor to protect mutable cache state while async functions perform network-like work. The cache can be called from concurrent tasks without exposing a mutable dictionary directly. Foundation provides `URL`, `Data`, and `URLSession`.

```swift
import Foundation

actor ImageCache {
    private var storage: [URL: Data] = [:]

    func data(for url: URL) -> Data? {
        storage[url]
    }

    func insert(_ data: Data, for url: URL) {
        storage[url] = data
    }
}

func fetchData(from url: URL, cache: ImageCache) async throws -> Data {
    if let cached = await cache.data(for: url) {
        return cached
    }

    let (data, _) = try await URLSession.shared.data(from: url)
    await cache.insert(data, for: url)
    return data
}
```

## Vocabulary

**Async function**: A function marked `async` that can suspend while waiting for work.

---

**`await`**: Marks a suspension point where the current task may pause and other work may run.

---

**Task**: A unit of asynchronous work managed by Swift's concurrency runtime.

---

**Structured concurrency**: A model where child tasks are scoped to the lifetime of their parent task.

---

**Actor**: A reference type that protects its mutable state by serializing access through actor isolation.

---

**Main actor**: The global actor for main-thread UI work.

---

**`Sendable`**: A marker for values that can safely cross concurrency domains.

---

**Data race**: Concurrent access to the same mutable state where at least one access writes and no synchronization protects it.

## Intuition

Concurrency has two different problems: waiting and sharing. `async`/`await` solves waiting. Actors and `Sendable` solve sharing. Many bugs happen when developers learn only the first half.

Swift wants you to isolate mutation. Instead of sharing a mutable dictionary across queues and hoping every access is locked, put the dictionary behind an actor. Instead of passing non-thread-safe reference types into detached work, make the boundary explicit.

## Async and Await

An async function can suspend. Suspension is not the same as blocking a thread. The task pauses and the executor can run other work. This standalone sample imports Foundation for networking and URL handling.

```swift
import Foundation

func loadUserName(id: String) async throws -> String {
    let url = URL(string: "https://example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return String(decoding: data, as: UTF8.self)
}
```

Every `await` is a point where the world can change. Do not assume state before an `await` is still true after it unless your isolation model guarantees it.

## Task Groups

Use task groups when you need concurrent child tasks with a clear parent scope. This keeps cancellation and error propagation understandable. The example imports Foundation for networking types.

```swift
import Foundation

func loadAll(_ urls: [URL]) async throws -> [Data] {
    try await withThrowingTaskGroup(of: Data.self) { group in
        for url in urls {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                return data
            }
        }

        var results: [Data] = []
        for try await data in group {
            results.append(data)
        }
        return results
    }
}
```

## Actors

Actors protect their isolated state. Code outside the actor must `await` when calling isolated methods because the call may need to hop to the actor's executor.

```swift
actor Counter {
    private var value = 0

    func increment() -> Int {
        value += 1
        return value
    }
}

let counter = Counter()
let value = await counter.increment()
print(value)
```

## Main Actor

UI state belongs on the main actor. In SwiftUI and modern Apple frameworks, many UI-facing types are expected to be main-actor isolated.

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var name = "Loading"

    func updateName(_ name: String) {
        self.name = name
    }
}
```

## Sendable

`Sendable` tells the compiler a value can cross concurrency boundaries safely. Value types whose stored values are sendable often get this automatically. Mutable reference types usually need redesign, actor isolation, or explicit unsafe promises.

```swift
struct SearchRequest: Sendable {
    let query: String
    let limit: Int
}
```

> [!WARNING]
> `@unchecked Sendable` is a promise to the compiler, not a fix. Use it only after you have a real synchronization strategy or immutable reference behavior.

## Swift 6 and Migration

Swift 6 language mode turns full data-race safety checking into a core contract. Migrating older code often means adding actor annotations, replacing shared mutable state, tightening closure captures, and marking safe value types as `Sendable`.

The practical migration order is:

1. Enable stricter concurrency diagnostics.
2. Fix obvious main-actor UI violations.
3. Move shared mutable state behind actors or locks.
4. Make boundary DTOs immutable and `Sendable`.
5. Audit every `@unchecked Sendable`.

## Pitfalls

- **Using `Task.detached` casually**: Detached tasks do not inherit actor context the way structured child tasks do.
- **Assuming `await` means background thread**: It means possible suspension, not a guaranteed thread hop.
- **Mutating actor state across await points without thinking**: State can change between awaits.
- **Annotating everything `@MainActor`**: It is useful for UI, but overuse can serialize work that should be concurrent.
- **Adding `@unchecked Sendable` to silence errors**: That removes compiler help at exactly the boundary where you need it.

## Exercises

1. Write an actor-backed counter and call it from ten child tasks.
2. Convert a callback-based function into an async function.
3. Create a `Sendable` request struct and explain why every stored property is safe.
4. Find a shared mutable dictionary in code and sketch how an actor would protect it.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- https://www.swift.org/migration/
- https://www.swift.org/blog/announcing-swift-6/
- https://www.swift.org/blog/swift-6.2-released/
- https://www.swift.org/blog/swift-6.3-released/
- Conversation with user on 2026-06-07

## Related

- Previous: [06 - Memory, ARC, Value Semantics, and Copy-on-Write](./06-memory-arc-value-semantics-and-copy-on-write.md)
- Next: [08 - SwiftPM, Testing, Compiling, and Shipping](./08-swiftpm-testing-compiling-and-shipping.md)
- Later: [10 - Senior-Level Swift Engineering Habits](./10-senior-level-swift-engineering-habits.md)
