---
title: "17 - Testing, Fuzzing, Vet, Race Detector, and CI"
created: 2026-06-07
updated: 2026-06-07
tags: [golang, programming-languages, testing, fuzzing, ci]
aliases: []
---

# 17 - Testing, Fuzzing, Vet, Race Detector, and CI

[toc]

> **TL;DR:** Go's standard toolchain includes tests, benchmarks, examples, fuzzing, vet analyzers, coverage, and the race detector. A production Go engineer should make `go test ./...`, `go test -race`, fuzz targets, benchmarks, and `go vet` routine.

## Real-World Example

This fuzz test verifies that parsing never panics and that valid round-trips stay stable. Fuzzing explores edge cases humans miss.

```go
package slug

import "testing"

func Slug(s string) string {
    // Pretend this normalizes a user-visible slug.
    if s == "" {
        return "empty"
    }
    return s
}

func FuzzSlug(f *testing.F) {
    f.Add("hello")
    f.Add("")
    f.Add("emoji-🙂")

    f.Fuzz(func(t *testing.T, input string) {
        got := Slug(input)
        if got == "" {
            t.Fatalf("empty slug for input %q", input)
        }
    })
}
```

Run it with a time limit.

```bash
go test -fuzz=FuzzSlug -fuzztime=30s ./...
```

## Vocabulary

**Table test**: A test loop over named cases.

---

**Subtest**: A named nested test created with `t.Run`.

---

**Benchmark**: A function named `BenchmarkXxx` that measures speed and allocations.

---

**Fuzz test**: A function named `FuzzXxx` that mutates inputs to discover failures.

---

**Race detector**: Runtime instrumentation that reports data races.

---

**Vet analyzer**: Static checker for suspicious but legal Go code.

## Intuition

Go tests are just Go code. That is the strength: no magic assertion framework is required. The risk is underusing the toolchain because it looks simple. Fuzzing, race detection, coverage, benchmarks, and vet all catch different classes of bugs.

Use table tests for known behavior, fuzzing for unknown edge cases, benchmarks for measured performance, and race detection for shared-memory concurrency.

## Table Tests

Table tests keep cases explicit and readable.

```go
func TestSlug(t *testing.T) {
    tests := []struct {
        name  string
        input string
        want  string
    }{
        {name: "empty", input: "", want: "empty"},
        {name: "word", input: "hello", want: "hello"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := Slug(tt.input); got != tt.want {
                t.Fatalf("Slug(%q) = %q, want %q", tt.input, got, tt.want)
            }
        })
    }
}
```

## Benchmarks

Benchmarks should report allocations when performance matters.

```go
func BenchmarkSlug(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = Slug("hello world")
    }
}
```

```bash
go test -bench=. -benchmem ./...
```

## Race Detector

Run the race detector on concurrent packages and integration tests.

```bash
go test -race ./...
```

The race detector slows programs down, so it is a CI/test tool, not a production runtime mode.

## Vet and Go 1.25

`go vet` catches suspicious constructs. Go 1.25 added analyzers for misplaced `sync.WaitGroup.Add` calls and IPv6-unsafe host/port formatting.

```bash
go vet ./...
```

## CI Baseline

A strong Go CI path:

```bash
go mod tidy
git diff --exit-code go.mod go.sum
go test ./...
go test -race ./...
go test -bench=. -benchmem ./...   # scheduled or performance branch
go vet ./...
govulncheck ./...
```

## Pitfalls

- **Loop variable capture**: Use modern Go semantics, but still understand closure capture in subtests and goroutines.
- **Race detector skipped forever**: Race bugs are often invisible in normal tests.
- **Benchmarks with dead-code elimination**: Ensure the result is used.
- **Fuzzing without seed cases**: Seeds guide exploration into meaningful states.
- **CI not checking `go mod tidy`**: Dependency drift becomes noise.

## Exercises

1. Add table tests for a parser.
2. Add a fuzz target for the parser.
3. Write a benchmark and inspect `B/op` and `allocs/op`.
4. Create a race intentionally and catch it with `go test -race`.
5. Run `go vet` and explain any findings.

## Sources

- https://go.dev/doc/security/fuzz/
- https://pkg.go.dev/testing
- https://go.dev/doc/articles/race_detector
- https://pkg.go.dev/cmd/vet
- https://go.dev/doc/go1.25
- https://go.dev/doc/tutorial/govulncheck
- Conversation with user on 2026-06-07

## Related

- Previous: [16 - Toolchain, Modules, Builds, and Release Engineering](./16-toolchain-modules-builds-and-release-engineering.md)
- Earlier: [8 - Concurrency Patterns and the Race Detector](./8-concurrency-patterns.md)
- Earlier: [10 - The Standard Library Tour](./10-standard-library-tour.md)
- Next: [18 - Reflection, Unsafe, cgo, and System Boundaries](./18-reflection-unsafe-cgo-and-system-boundaries.md)

