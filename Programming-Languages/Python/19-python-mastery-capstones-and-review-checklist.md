---
title: "19 - Python Mastery Capstones and Review Checklist"
created: 2026-06-07
updated: 2026-06-07
tags: [python, programming-languages, mastery, capstone]
aliases: []
---

# 19 - Python Mastery Capstones and Review Checklist

[toc]

> **TL;DR:** You master Python by connecting runtime internals to shipped software. Build a CLI, a package, an async service, a profiling lab, a native-boundary wrapper, and a testing harness; each project should force you to explain the object model, import system, memory behavior, concurrency, packaging, and deployment.

## Real-World Example

This small domain model is a capstone seed. It can become a CLI, service, package, or async worker without changing the core model.

```python
from dataclasses import dataclass
from enum import StrEnum


class JobState(StrEnum):
    QUEUED = "queued"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"


@dataclass(slots=True)
class Job:
    id: str
    state: JobState = JobState.QUEUED

    def start(self) -> None:
        if self.state is not JobState.QUEUED:
            raise ValueError(f"cannot start from {self.state}")
        self.state = JobState.RUNNING
```

## Vocabulary

**Capstone**: A project that forces multiple layers of knowledge to interact.

---

**Invariant**: A rule that must stay true, such as "only queued jobs can start."

---

**Runtime hypothesis**: A claim about Python behavior that you verify with code, `dis`, profiling, or source/docs.

---

**Production gate**: A check that must pass before shipping.

## Intuition

The difference between "knows Python" and "masters Python" is the ability to predict consequences. If you add a decorator, what object is created? If you add a closure, what does it capture? If you add a dependency, how is it resolved and shipped? If you add threads, what happens under the GIL or free-threaded build?

Use these capstones to make the invisible layers visible.

## Capstone 1: CLI Package

Build a real CLI with `argparse` or Typer.

Requirements:

- `pyproject.toml` with console script.
- `src` layout.
- Tests for parsing and behavior.
- `python -m package` support.
- Wheel build and install into a clean venv.

Concepts proven:

- Import system.
- Packaging.
- Entry points.
- Error handling.
- Testability.

## Capstone 2: Async Service

Build a small FastAPI or aiohttp service.

Requirements:

- Pydantic or dataclass boundary models.
- Async database or fake repository.
- Structured logs.
- Timeouts.
- Graceful shutdown.
- Integration tests.

Concepts proven:

- Asyncio.
- Context managers.
- Serialization.
- Observability.
- Deployment.

## Capstone 3: Runtime Lab

Create experiments that prove how Python works.

Experiments:

- Disassemble loops, comprehensions, and method calls.
- Compare object sizes with and without `__slots__`.
- Use `tracemalloc` snapshots.
- Create and fix a reference cycle.
- Inspect `sys.modules` during circular imports.

Concepts proven:

- Bytecode.
- Object layout.
- Reference counting.
- GC.
- Imports.

## Capstone 4: Native Boundary

Wrap a small C function or native library.

Requirements:

- Safe Python API.
- Explicit argument and return types.
- Error conversion.
- Tests for invalid inputs.
- Notes on ownership and thread-safety.

Concepts proven:

- FFI.
- ABI.
- Memory ownership.
- Free-threaded readiness.

## Review Checklist

Use this checklist when reading Python code:

- Are mutable defaults avoided?
- Are imports side-effect-light?
- Does package execution work with `python -m`?
- Are public types and functions typed enough for callers?
- Are exceptions specific and recoverable?
- Are async calls awaited and bounded by timeouts?
- Are blocking calls kept out of the event loop?
- Are tests deterministic?
- Are profiles based on representative input?
- Are object lifetimes and cycles understood?
- Are dependencies pinned and scanned?
- Is the deployment path repeatable?

## Pitfalls

- **Only learning syntax**: Syntax is the smallest part of Python mastery.
- **Treating CPython details as language guarantees**: Know which facts are CPython-specific.
- **No clean packaging path**: A script is not a shipped product.
- **Ignoring native extensions**: Many Python performance and deployment issues come from compiled dependencies.
- **No operational thinking**: Logs, config, timeouts, and rollbacks are part of correctness.

## Exercises

1. Pick one capstone and write its acceptance checklist.
2. Add `dis` output to a runtime lab note and explain each surprising opcode.
3. Build a wheel, install it in a fresh environment, and run tests against the installed package.
4. Create a circular import intentionally, then refactor it away.
5. Run the same threaded workload under regular CPython and a free-threaded build if available.

## Sources

- https://docs.python.org/3.14/
- https://docs.python.org/3/whatsnew/3.14.html
- https://docs.python.org/3/howto/free-threading-python.html
- https://packaging.python.org/
- https://docs.python.org/3.14/library/dis.html
- Conversation with user on 2026-06-07

## Related

- Previous: [18 - Deployment, Operations, Security, and Release Engineering](./18-deployment-operations-security-and-release-engineering.md)
- Start over: [1 - What is Python](./1-what-is-python.md)
- Earlier: [13 - Memory Model and PyObject Layout](./13-memory-model-and-pyobject-layout.md)
- Earlier: [15 - CPython Compiler, Bytecode, and Import System](./15-cpython-compiler-bytecode-import-system.md)

