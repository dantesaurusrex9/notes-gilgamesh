---
title: "11 - Standard Library, Strings, Collections, and Algorithms"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, standard-library, collections]
aliases: []
---

# 11 - Standard Library, Strings, Collections, and Algorithms

[toc]

> **TL;DR:** Swift's standard library is small but deep. Master `String`, `Array`, `Dictionary`, `Set`, `Sequence`, `Collection`, lazy operations, sorting, hashing, and Unicode correctness before reaching for third-party dependencies.

## Real-World Example

This example normalizes words from user input and counts frequency. It uses strings, sequences, dictionaries, sorting, and a small amount of Unicode-aware filtering.

```swift
let text = "Swift, swift! Strings are Unicode-aware."

let counts = text
    .lowercased()
    .split { !$0.isLetter }
    .reduce(into: [Substring: Int]()) { result, word in
        result[word, default: 0] += 1
    }

let sorted = counts.sorted { lhs, rhs in
    if lhs.value == rhs.value {
        return lhs.key < rhs.key
    }
    return lhs.value > rhs.value
}

for (word, count) in sorted {
    print("\(word): \(count)")
}
```

## Vocabulary

**Standard library**: The core types and functions available without importing Foundation.

---

**Sequence**: A type that can be iterated once or many times, depending on implementation.

---

**Collection**: A sequence with stable indices and traversal guarantees.

---

**Lazy sequence**: A wrapper that defers transformations until elements are requested.

---

**Substring**: A view into a larger string's storage. Convert to `String` when storing long-term.

---

**Unicode scalar**: A single Unicode code point. User-visible characters can contain multiple scalars.

## Intuition

Swift strings are not byte arrays. A user-visible character can be made from multiple Unicode scalars, so indexing by integer offset is intentionally not the normal model. This protects correctness for names, emoji, accents, and non-English text.

Collections are value types with copy-on-write storage. You can pass arrays and dictionaries around freely in most code, but hot paths still need measurement. The high-level API is expressive; the low-level cost depends on allocation, hashing, sorting, bridging, and mutation.

## Strings and Substrings

`String.split` returns `Substring` values because they can share storage with the original string. That is efficient for short pipelines, but storing many substrings can keep the original large string alive.

```swift
let logLine = "2026-06-07 INFO request finished"
let pieces = logLine.split(separator: " ")

let level = String(pieces[1])
print(level)
```

> [!TIP]
> Keep `Substring` during parsing pipelines. Convert to `String` when you store the result beyond the lifetime of the original input.

## Arrays

Arrays are ordered collections. Appending is usually amortized constant time, but inserting or removing near the front shifts elements.

```swift
var queue = ["first", "second"]
queue.append("third")

let next = queue.removeFirst()
print(next)
```

For a real high-throughput FIFO queue, avoid repeated `removeFirst()` on large arrays. Use an index offset, a ring buffer, or a dedicated deque implementation.

## Dictionaries and Sets

Dictionaries and sets depend on hashing. Keys must be `Hashable`, and good domain modeling often means creating small typed keys instead of passing raw strings everywhere.

```swift
struct UserID: Hashable {
    let rawValue: String
}

var activeUsers: Set<UserID> = []
activeUsers.insert(UserID(rawValue: "u-1"))
```

## Lazy Pipelines

Use `lazy` when a transformation chain should not allocate intermediate arrays. It is most useful when you stop early or process large collections.

```swift
let firstLargeSquare = (1...)
    .lazy
    .map { $0 * $0 }
    .first { $0 > 10_000 }

print(firstLargeSquare ?? 0)
```

## Sorting and Stability

Swift's `sorted()` returns a new array. `sort()` mutates in place. Do not assume sort stability unless the API promises it for your context.

```swift
var scores = [80, 100, 95]
scores.sort()
print(scores)
```

## Pitfalls

- **Integer-indexing strings**: Swift strings use `String.Index`, not integer offsets, because Unicode is not fixed width.
- **Long-lived substrings**: A small substring can keep a large original string alive.
- **Hidden intermediate arrays**: Chained `map` and `filter` can allocate. Use `lazy` when it matters.
- **Dictionary order assumptions**: Do not write business logic that depends on dictionary iteration order.
- **Premature micro-optimization**: Standard library operations are optimized. Profile before replacing them.

## Exercises

1. Count word frequency in a paragraph and return the top five words.
2. Rewrite a `map().filter().first` chain with `lazy` and explain what allocation changes.
3. Create typed IDs for `User`, `Order`, and `Invoice` and store them in dictionaries.
4. Parse a string with accents or emoji and explain why character count differs from byte count.

## Sources

- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/
- https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/
- Conversation with user on 2026-06-07

## Related

- Previous: [10 - Senior-Level Swift Engineering Habits](./10-senior-level-swift-engineering-habits.md)
- Next: [12 - Initialization, Methods, Properties, and Subscripts](./12-initialization-methods-properties-and-subscripts.md)
- Earlier: [06 - Memory, ARC, Value Semantics, and Copy-on-Write](./06-memory-arc-value-semantics-and-copy-on-write.md)

