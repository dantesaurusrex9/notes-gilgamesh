---
title: "20 - Go Mastery Capstones and Review Checklist"
created: 2026-06-07
updated: 2026-06-07
tags: [golang, programming-languages, mastery, capstone]
aliases: []
---

# 20 - Go Mastery Capstones and Review Checklist

[toc]

> **TL;DR:** Go mastery means connecting simple syntax to production systems: packages, interfaces, memory, concurrency, testing, profiling, release binaries, and operational behavior. Build capstones that force every layer to work together.

## Real-World Example

This small domain model is a capstone seed. It can power a CLI, HTTP service, worker, or library without depending on a framework.

```go
package jobs

import "errors"

type State string

const (
    Queued    State = "queued"
    Running   State = "running"
    Succeeded State = "succeeded"
    Failed    State = "failed"
)

type Job struct {
    ID    string
    State State
}

func (j *Job) Start() error {
    if j.State != Queued {
        return errors.New("job must be queued before start")
    }
    j.State = Running
    return nil
}
```

## Vocabulary

**Capstone**: A project that proves several skills at once.

---

**Invariant**: A rule that must remain true, such as "only queued jobs can start."

---

**Operational contract**: The runtime behavior a service promises: timeouts, shutdown, logs, metrics, limits, and recovery.

---

**Release artifact**: The binary, container, or module version users actually consume.

## Intuition

Go's simplicity can hide depth. The syntax is small, but the hard parts are design boundaries, goroutine lifetimes, memory pressure, interface size, package structure, and release discipline.

The goal is not to write clever Go. The goal is to write Go that another engineer can debug at 3 AM with logs, profiles, tests, and a clear package graph.

## Capstone 1: CLI Tool

Build a CLI that reads, validates, transforms, and writes data.

Requirements:

- `cmd/mytool/main.go` entry point.
- Internal package for business logic.
- Table tests for parsing.
- Fuzz test for input decoder.
- Release binaries for Linux and macOS.

Concepts proven:

- Package layout.
- Error handling.
- I/O and buffering.
- Tests and fuzzing.
- Cross-compilation.

## Capstone 2: HTTP Service

Build a service with graceful shutdown and observability.

Requirements:

- `net/http` or a small router.
- `context.Context` propagation.
- Timeouts on server and client.
- Structured logs with `log/slog`.
- Prometheus or runtime metrics.
- pprof protected from public access.
- Race detector clean.

Concepts proven:

- Concurrency.
- Context cancellation.
- Production service design.
- Runtime inspection.
- Deployment.

## Capstone 3: Concurrent Worker Pool

Build a worker pool that processes jobs with cancellation and backpressure.

Requirements:

- Bounded queue.
- `context.Context` cancellation.
- No goroutine leaks.
- Tests for cancellation and shutdown.
- Race detector clean.
- Benchmark for throughput.

Concepts proven:

- Goroutines.
- Channels.
- `sync`.
- Race detector.
- Benchmarks.

## Capstone 4: Low-Level Boundary

Build a safe wrapper around an unsafe or cgo operation.

Requirements:

- Unsafe code isolated to one package.
- Safe public API.
- Ownership rules documented.
- Tests across edge cases.
- Build tags for platform-specific code.

Concepts proven:

- Reflection and unsafe tradeoffs.
- cgo build constraints.
- Portability.
- Manual review.

## Review Checklist

Use this checklist when reviewing Go:

- Is the package boundary small and coherent?
- Are interfaces accepted at consumers instead of exported prematurely?
- Are errors wrapped with context and checked with `errors.Is` or `errors.As`?
- Does every goroutine have a shutdown path?
- Are channels closed by senders only?
- Is shared state protected or avoided?
- Are contexts passed to blocking work?
- Are tests table-driven where useful?
- Does fuzzing cover parsers and decoders?
- Does `go test -race ./...` pass?
- Are allocations measured in hot paths?
- Can you inspect the release binary with `go version -m`?
- Is cgo or unsafe isolated and documented?

## Pitfalls

- **Goroutine leaks**: A goroutine without cancellation is a memory leak with a stack.
- **Huge interfaces**: Small interfaces compose; large interfaces freeze design.
- **Panics for normal errors**: Return errors for expected failures.
- **Unbounded queues**: They hide backpressure until memory explodes.
- **No profiling path**: If you cannot profile it, you cannot operate it confidently.

## Exercises

1. Pick one capstone and write its acceptance checklist.
2. Add a fuzz target to a parser.
3. Add graceful shutdown to an HTTP server.
4. Capture a CPU profile under load.
5. Build a release binary and inspect it with `go version -m`.

## Sources

- https://go.dev/doc/effective_go
- https://go.dev/ref/mod
- https://go.dev/doc/security/fuzz/
- https://go.dev/doc/pgo
- https://go.dev/doc/go1.25
- Conversation with user on 2026-06-07

## Related

- Previous: [19 - Profiling, Tracing, PGO, and Runtime Tuning](./19-profiling-tracing-pgo-and-runtime-tuning.md)
- Start over: [1 - What is Go](./1-what-is-go.md)
- Earlier: [7 - Goroutines and Channels](./7-goroutines-and-channels.md)
- Earlier: [16 - Toolchain, Modules, Builds, and Release Engineering](./16-toolchain-modules-builds-and-release-engineering.md)

