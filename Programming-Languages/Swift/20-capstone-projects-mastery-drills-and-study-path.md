---
title: "20 - Capstone Projects, Mastery Drills, and Study Path"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, capstone, mastery]
aliases: []
---

# 20 - Capstone Projects, Mastery Drills, and Study Path

[toc]

> **TL;DR:** You master Swift by building progressively harder systems: a CLI, a package, a tested library, a SwiftUI app, a server API, and a low-level interop wrapper. Each project should force syntax, types, memory, concurrency, testing, performance, and shipping to connect.

## Real-World Example

This capstone starter models a small task tracker domain without UI or storage. It is intentionally plain Swift so it can become a CLI, SwiftUI app, or server API later.

```swift
struct TaskID: Hashable, Sendable {
    let rawValue: String
}

enum TaskStatus: Sendable {
    case open
    case inProgress
    case done
}

struct TaskItem: Equatable, Sendable {
    let id: TaskID
    var title: String
    var status: TaskStatus
}

struct TaskList {
    private(set) var tasks: [TaskID: TaskItem] = [:]

    mutating func add(_ task: TaskItem) {
        tasks[task.id] = task
    }

    mutating func markDone(id: TaskID) {
        guard var task = tasks[id] else { return }
        task.status = .done
        tasks[id] = task
    }
}
```

## Vocabulary

**Capstone**: A project that forces multiple skills to work together.

---

**Drill**: A focused repetition exercise for one skill.

---

**Milestone**: A verifiable checkpoint in a project.

---

**Refactoring pass**: A deliberate change that improves structure while preserving behavior.

---

**Release candidate**: A build that could ship if final verification passes.

## Intuition

Reading gives vocabulary. Building gives judgment. You will not internalize Swift's memory model until you debug a retain cycle. You will not respect actor isolation until you hit a concurrency warning. You will not understand signing until you archive and distribute an app.

The study path should make hidden assumptions visible. Every capstone has a test gate and a shipping gate. That is what turns language knowledge into engineering skill.

## Project 1: CLI Tool

Build a command-line task tracker.

Milestones:

1. SwiftPM executable package.
2. Typed domain model.
3. File-backed JSON persistence.
4. Argument parsing.
5. Tests for add, list, complete, and invalid input.
6. Release build artifact.

Skills practiced:

- Syntax and optionals.
- Codable.
- Error handling.
- SwiftPM.
- Tests.
- Small API design.

## Project 2: Reusable Library

Extract the domain into a package library.

Milestones:

1. Public types documented with DocC comments.
2. Internal storage hidden.
3. Semantic version tag.
4. Behavior tests and failure tests.
5. README with usage example.

Skills practiced:

- Access control.
- Modules.
- Public API design.
- Documentation.
- Versioning.

## Project 3: SwiftUI App

Build a small desktop or iOS app on top of the library.

Milestones:

1. SwiftUI list and detail view.
2. Main-actor model.
3. State-driven navigation.
4. Local persistence.
5. Unit tests for model logic.
6. Archive and TestFlight or local macOS distribution path.

Skills practiced:

- State ownership.
- Main actor.
- UI lifecycle.
- App signing.
- Release checklist.

## Project 4: Server API

Expose the same task model through a server.

Milestones:

1. Vapor or Hummingbird endpoint.
2. JSON request and response types.
3. Validation errors.
4. Structured request logging.
5. Docker release build.
6. Linux test path.

Skills practiced:

- Async request handling.
- Server-side Swift.
- Observability.
- Container packaging.
- Linux compatibility.

## Project 5: Low-Level Wrapper

Write a tiny safe wrapper around an unsafe byte-buffer function.

Milestones:

1. Safe Swift API.
2. Unsafe scope contained to one file.
3. Tests for empty, small, and large inputs.
4. AddressSanitizer run.
5. Documentation of ownership and lifetime rules.

Skills practiced:

- Unsafe pointers.
- Memory safety.
- Error handling.
- Testing dangerous boundaries.

## Weekly Study Loop

Use this loop until the language feels natural:

1. Read one note.
2. Type every code sample manually.
3. Write one variation without looking.
4. Add one test.
5. Break it intentionally and read the compiler error.
6. Fix it.
7. Summarize the rule in your own words.

## Mastery Drills

- Rewrite optional-heavy code with `guard`.
- Replace booleans with enums.
- Find and fix a retain cycle.
- Convert a callback to async/await.
- Add `Sendable` to a value boundary.
- Split an executable into library plus thin `main`.
- Add DocC comments to public API.
- Profile release-mode code before optimizing.
- Archive an app and document each signing step.
- Containerize a server in release mode.

## Pitfalls

- **Only reading**: Swift must be typed and debugged to stick.
- **Skipping release workflows**: Shipping teaches constraints tutorials ignore.
- **Avoiding compiler errors**: The errors are part of the teacher.
- **Building only UI**: You need CLI, library, server, and low-level practice too.
- **No tests**: Without tests, refactoring does not build confidence.

## Exercises

1. Start Project 1 and create the SwiftPM package.
2. Implement `TaskID`, `TaskStatus`, and `TaskItem`.
3. Add one Swift Testing test for `markDone`.
4. Write a checklist for when the CLI is "done enough" to release.

## Sources

- https://www.swift.org/getting-started/
- https://docs.swift.org/swiftpm/documentation/packagemanagerdocs/
- https://www.swift.org/documentation/api-design-guidelines/
- https://www.swift.org/documentation/server/guides/
- https://developer.apple.com/documentation/xcode/distributing-your-app-for-beta-testing-and-releases
- Conversation with user on 2026-06-07

## Related

- Previous: [19 - Advanced Testing: Fakes, Async, UI Boundaries, and CI](./19-advanced-testing-fakes-async-ui-boundaries-and-ci.md)
- Start over: [01 - Swift Mastery Map and Current Toolchain](./01-swift-mastery-map-and-current-toolchain.md)
- Earlier: [10 - Senior-Level Swift Engineering Habits](./10-senior-level-swift-engineering-habits.md)

