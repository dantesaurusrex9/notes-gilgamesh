---
title: "4 - Functions, Closures, Decorators"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 4 - Functions, Closures, Decorators

[toc]

> **TL;DR:** In Python, functions are first-class objects — they can be passed, returned, stored in containers, and have attributes. Closures bind free variables from an enclosing scope into the function object, enabling stateful callbacks and factory functions. Decorators are functions that take a function and return a (usually enhanced) function, implemented with syntactic sugar via `@`; `functools.wraps` preserves the original function's metadata through the wrapping.

## Vocabulary

**First-class function**: A function that can be treated as a value — assigned to a variable, passed as an argument, returned from another function, stored in a data structure. Python functions are first-class objects of type `function`.

---

**Closure**: A function that retains references to names in its enclosing scope even after that scope has returned. The enclosing scope's locals are captured in the function's `__closure__` attribute as `cell` objects.

---

**Free variable**: A variable referenced inside a function but not defined in that function's local scope. Free variables are what a closure captures.

---

**`nonlocal`**: A declaration that tells Python to look for a name in the enclosing scope (not global). Required when you want to *assign* to a free variable, not just read it.

---

**Default argument**: A value bound to a parameter at function definition time and used when the caller omits that argument. Stored in `func.__defaults__` (positional) and `func.__kwdefaults__` (keyword-only).

---

**`*args`**: Collects extra positional arguments into a tuple. The `*` can appear at most once in a signature.

---

**`**kwargs`**: Collects extra keyword arguments into a dict. Appears after `*args` if both are present.

---

**Positional-only parameter** (`/`): Parameters before `/` in the signature that callers cannot pass by name. Enforced since Python 3.8 (PEP 570). Example: `def f(x, /, y): ...` — `x` is positional-only.

---

**Keyword-only parameter** (`*`): Parameters after a bare `*` or after `*args` that callers must pass by name. Example: `def f(x, *, y): ...` — `y` is keyword-only.

---

**Decorator**: A callable that takes a callable and returns a callable. Applied with `@decorator` syntax above a function or class definition. Equivalent to `func = decorator(func)`.

---

**`functools.wraps`**: A decorator that copies `__name__`, `__doc__`, `__annotations__`, `__qualname__`, and `__module__` from the wrapped function onto the wrapper, and sets `__wrapped__` to the original.

---

**Parameterised decorator**: A decorator factory — a function that accepts arguments and returns a decorator. Requires three levels of nesting.

---

## Intuition

A function definition in Python creates a function object and binds it to a name. The function object carries the compiled bytecode (`__code__`), default values (`__defaults__`), the global namespace it was defined in (`__globals__`), and if it is a closure, the captured cell objects (`__closure__`). Nothing about this is special-cased — functions are objects just like lists and integers.

Closures solve the "how do I make a function that remembers something?" problem without requiring a class. A closure is a lightweight object with one method and some state. Decorators solve the "how do I add behaviour to a function without modifying its body?" problem — logging, timing, authentication, retry, caching — all expressed as wrappers.

## Default Arguments

### The Mutable Default Trap (Redux)

Default arguments are evaluated once, at function definition time. If the default is mutable, it is shared across all calls — this is the most famous Python footgun.

```python
# WRONG: list is created once and shared
def bad_extend(item: int, lst: list[int] = []) -> list[int]:
    lst.append(item)
    return lst

print(bad_extend(1))  # >>> [1]
print(bad_extend(2))  # >>> [1, 2]  — not [2]!

# The default lives in bad_extend.__defaults__[0] — same object every call
print(bad_extend.__defaults__)  # >>> ([1, 2],)
```

```python
# RIGHT: use None as sentinel
def good_extend(item: int, lst: list[int] | None = None) -> list[int]:
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

There is a legitimate use for mutable defaults: manual caching (though `functools.lru_cache` is better):

```python
def cached_fib(n: int, _cache: dict[int, int] = {}) -> int:
    if n in _cache:
        return _cache[n]
    if n <= 1:
        return n
    result = cached_fib(n - 1, _cache) + cached_fib(n - 2, _cache)
    _cache[n] = result
    return result
```

## `*args`, `**kwargs`, and Parameter Kinds

Python's parameter syntax is rich. All five kinds can appear in one signature, in a specific order.

```python
def full_sig(
    pos_only: int,          # positional-only (before /)
    /,
    normal: str,            # positional-or-keyword
    *args: float,           # extra positionals → tuple
    kw_only: bool,          # keyword-only (after *)
    **kwargs: object,       # extra keywords → dict
) -> None:
    print(pos_only, normal, args, kw_only, kwargs)

full_sig(1, "a", 3.0, 4.0, kw_only=True, x=99)
# >>> 1 a (3.0, 4.0) True {'x': 99}
```

Unpacking arguments from a container uses `*` and `**`:

```python
def add(a: int, b: int, c: int) -> int:
    return a + b + c

args = (1, 2, 3)
kwargs = {"a": 1, "b": 2, "c": 3}
print(add(*args))     # >>> 6
print(add(**kwargs))  # >>> 6
```

## Closures and `nonlocal`

### Reading a Free Variable

A closure can read a free variable without any special declaration:

```python
from collections.abc import Callable


def make_multiplier(factor: float) -> Callable[[float], float]:
    """Returns a function that multiplies its argument by factor."""
    def multiplier(x: float) -> float:
        return x * factor   # factor is a free variable, captured from enclosing scope
    return multiplier

triple = make_multiplier(3.0)
print(triple(7.0))    # >>> 21.0
print(triple.__closure__)   # >>> (<cell at 0x...>,)
print(triple.__closure__[0].cell_contents)  # >>> 3.0
```

### Assigning to a Free Variable

Attempting to assign to a free variable creates a local variable instead, causing an `UnboundLocalError`. Use `nonlocal` to tell Python the name lives in the enclosing scope.

```python
from collections.abc import Callable


def make_counter() -> tuple[Callable[[], int], Callable[[], None]]:
    count: int = 0

    def get() -> int:
        return count     # reading — no declaration needed

    def increment() -> None:
        nonlocal count   # must declare; without this, `count += 1` fails
        count += 1

    return get, increment

get, inc = make_counter()
inc(); inc(); inc()
print(get())  # >>> 3
```

> [!WARNING]
> Without `nonlocal count`, the line `count += 1` inside `increment` is compiled as "load local `count`, add 1, store local `count`." Because Python sees an assignment to `count` anywhere in the function body, it marks `count` as local for the entire function. The load then fails with `UnboundLocalError: local variable 'count' referenced before assignment` — even though a `count` in the enclosing scope exists. This is one of the most confusing error messages for Python newcomers.

## Decorators

### The Core Pattern

A decorator is a function that takes a function and returns a function. The `@` syntax is sugar.

```python
import time
from collections.abc import Callable
from typing import TypeVar
import functools

F = TypeVar("F", bound=Callable[..., object])


def timer(func: F) -> F:
    """Decorator that prints how long the function took."""
    @functools.wraps(func)
    def wrapper(*args: object, **kwargs: object) -> object:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper  # type: ignore[return-value]


@timer
def slow_sum(n: int) -> int:
    return sum(range(n))

result = slow_sum(10_000_000)
# >>> slow_sum took 0.2341s
```

`@functools.wraps(func)` is not optional in production code — without it, `wrapper.__name__` is `"wrapper"`, which breaks doctest, pytest introspection, logging, and stack traces.

### Parameterised Decorators

When the decorator itself needs arguments, you add one more level of nesting: a factory function that returns the decorator.

```python
import functools
from collections.abc import Callable
from typing import TypeVar

F = TypeVar("F", bound=Callable[..., object])


def retry(max_attempts: int = 3, exceptions: tuple[type[Exception], ...] = (Exception,)) -> Callable[[F], F]:
    """Decorator factory: retry the function up to max_attempts times on failure."""
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args: object, **kwargs: object) -> object:
            last_exc: Exception | None = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    last_exc = exc
                    print(f"Attempt {attempt}/{max_attempts} failed: {exc}")
            raise RuntimeError(
                f"{func.__name__} failed after {max_attempts} attempts"
            ) from last_exc
        return wrapper  # type: ignore[return-value]
    return decorator


@retry(max_attempts=3, exceptions=(ValueError, IOError))
def flaky_operation(n: int) -> int:
    import random
    if random.random() < 0.7:
        raise ValueError("transient error")
    return n * 2
```

> [!TIP]
> For production retry logic, use `tenacity` (third-party) — it handles exponential backoff, jitter, and deadline logic. But writing a `retry` decorator yourself is the canonical exercise for understanding parameterised decorators and closures simultaneously.

### Stacking Decorators

Multiple decorators apply bottom-up: `@a @b` means `func = a(b(func))`.

```python
import functools
from collections.abc import Callable
from typing import TypeVar

F = TypeVar("F", bound=Callable[..., object])


def bold(func: F) -> F:
    @functools.wraps(func)
    def wrapper(*a: object, **kw: object) -> object:
        return f"<b>{func(*a, **kw)}</b>"
    return wrapper  # type: ignore[return-value]


def italic(func: F) -> F:
    @functools.wraps(func)
    def wrapper(*a: object, **kw: object) -> object:
        return f"<i>{func(*a, **kw)}</i>"
    return wrapper  # type: ignore[return-value]


@bold
@italic
def greet(name: str) -> str:
    return f"Hello, {name}"

print(greet("World"))  # >>> <b><i>Hello, World</i></b>
# Applied: greet = bold(italic(greet))
```

## Real-world Example

A production-grade logging decorator with parameterisation, `functools.wraps`, and type safety — the kind used in a FastAPI service to instrument handler functions.

```python
import functools
import logging
import time
from collections.abc import Callable
from typing import TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")

logger = logging.getLogger(__name__)


def log_call(
    level: int = logging.INFO,
    include_args: bool = False,
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """
    Decorator factory: logs entry, exit, and duration of a function call.

    Args:
        level: logging level (default INFO).
        include_args: whether to log positional/keyword arguments.
    """
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            args_str = f"args={args!r} kwargs={kwargs!r}" if include_args else ""
            logger.log(level, "Calling %s %s", func.__qualname__, args_str)
            start = time.perf_counter()
            try:
                result = func(*args, **kwargs)
                elapsed = time.perf_counter() - start
                logger.log(level, "%s returned in %.4fs", func.__qualname__, elapsed)
                return result
            except Exception as exc:
                elapsed = time.perf_counter() - start
                logger.exception(
                    "%s raised %s after %.4fs",
                    func.__qualname__, type(exc).__name__, elapsed,
                )
                raise
        return wrapper
    return decorator


@log_call(level=logging.DEBUG, include_args=True)
def process_batch(items: list[int], batch_size: int = 32) -> list[int]:
    return [x * 2 for x in items]
```

The use of `ParamSpec` (PEP 612, Python 3.10+) preserves the full type signature of the decorated function so pyright knows the arguments and return type of `process_batch` after decoration.

> [!NOTE]
> `ParamSpec` (`from typing import ParamSpec`) is the correct way to annotate decorators that preserve the decorated function's signature. Without it, `wrapper` is typed as `(*args: Any, **kwargs: Any) -> Any`, losing all type information for callers. `ParamSpec` is supported in Python 3.10+ natively and in 3.9 via `from typing_extensions import ParamSpec`.

## In Practice

**`functools.lru_cache` and `functools.cache`** are the most-used decorators in the standard library. `cache` (Python 3.9+) is `lru_cache(maxsize=None)`. They use a dictionary keyed on the function arguments, so all arguments must be hashable.

**`functools.partial`** is a factory for pre-filling arguments. `partial(sorted, key=str.lower)` returns a new callable with `key` already bound. Use it to adapt function signatures for APIs that expect a specific arity.

**Closures leak memory if they capture large objects.** A closure keeps a reference to the captured cell, which keeps the cell's contents alive. If a closure captures a large tensor or database connection and the closure itself is stored in a long-lived container, the large object will not be freed. Use weak references (`weakref.ref`) when the captured object should not be kept alive solely by the closure.

> [!CAUTION]
> `eval()` and `exec()` inside a decorator can execute arbitrary code if the inputs come from user-supplied strings. Never use `eval`/`exec` with unvalidated input, even in a supposedly "safe" sandboxed decorator. There is no reliable Python sandbox — use subprocess isolation or a separate interpreter process.

## Pitfalls

- **"Decorators run at call time."** — No. Decorators run at *definition* time (when the `def` or `class` statement is executed, not when the function is called). `@lru_cache` attaches the cache at definition time; `@app.route("/")` registers the route at import time.
- **"Forgetting `functools.wraps` is cosmetic."** — It is not. Missing `functools.wraps` breaks `help()`, doctests, pytest output, logging, tracebacks, and any introspection that relies on `__name__` or `__doc__`. It also breaks decorator stacking with certain frameworks.
- **"`*args` captures keyword arguments."** — No. `*args` captures only extra *positional* arguments. `**kwargs` captures extra keyword arguments. `def f(*args): print(args); f(x=1)` raises `TypeError`.
- **"Closures capture the value, not the variable."** — Closures capture the *cell*, not the value at capture time. Classic trap: `lambdas = [lambda: i for i in range(5)]`. All lambdas return `4` because they share the same `i` cell, whose final value is 4. Fix: `lambda i=i: i` (default argument bakes in the current value).
- **"You can define the inner function before the outer returns."** — You can define it, but the closure only captures variables that are *in scope at definition time* in the outer function. Variables defined *after* the closure definition are not captured.

## Exercises

### Exercise 1 — Closure trap

What does the following print? Explain why.

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)

for f in funcs:
    print(f())
```

#### Solution

All three calls print `2`. All three lambdas close over the same `i` variable (the loop variable is a single name in the enclosing scope). By the time any lambda is called, the loop has finished and `i = 2`. The lambdas capture the *cell*, not the *value at capture time*.

**Fix**: Bake in the current value using a default argument:

```python
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)

for f in funcs:
    print(f())
# >>> 0 1 2
```

Default arguments are evaluated at `lambda` definition time, so each lambda gets its own copy of `i`'s current value.

---

### Exercise 2 — Write a `memoize` decorator

Implement a `memoize` decorator (without using `functools.lru_cache`) that caches results by arguments.

#### Solution

```python
import functools
from collections.abc import Callable
from typing import TypeVar, Any

R = TypeVar("R")


def memoize(func: Callable[..., R]) -> Callable[..., R]:
    """Cache results of func keyed by its arguments. All args must be hashable."""
    cache: dict[tuple[Any, ...], R] = {}

    @functools.wraps(func)
    def wrapper(*args: object) -> R:
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]

    # Expose cache for inspection/clearing
    wrapper.cache = cache  # type: ignore[attr-defined]
    return wrapper


@memoize
def fib(n: int) -> int:
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(30))           # >>> 832040
print(len(fib.cache))    # type: ignore[attr-defined]  # >>> 31
```

The cache is a dict in the closure, keyed on `args` (a tuple). Only positional arguments are supported (adding `**kwargs` requires `tuple(sorted(kwargs.items()))` as part of the key). The real `functools.lru_cache` adds a `maxsize` eviction policy using an ordered dict — this version has unbounded growth.

---

### Exercise 3 — `nonlocal` vs `global`

Explain the difference between `nonlocal` and `global`, and give a case where `nonlocal` is correct but `global` would be wrong.

#### Solution

`global x` makes `x` refer to the module-level global variable named `x`. `nonlocal x` makes `x` refer to the nearest enclosing scope's variable named `x` (not module-level).

`global` is wrong when you have multiple closures created from the same factory, each needing their own counter — `global` would make all counters share a single module-level variable:

```python
def make_counter() -> tuple[..., ...]:
    count = 0            # enclosing scope variable

    def get() -> int:
        return count

    def reset() -> None:
        nonlocal count   # correct: each factory call has its own count
        count = 0

    def inc() -> None:
        nonlocal count
        count += 1

    return get, reset, inc

get1, reset1, inc1 = make_counter()
get2, _, inc2 = make_counter()

inc1(); inc1()
inc2()
print(get1(), get2())  # >>> 2 1 — independent counters
```

If `inc` used `global count`, `count` would be a single module-level variable shared by both `make_counter()` instances.

---

### Exercise 4 — Implement a `once` decorator

Write a decorator `once` such that the decorated function executes at most once; subsequent calls return the cached result without re-executing the body.

#### Solution

```python
import functools
from collections.abc import Callable
from typing import TypeVar, Any

R = TypeVar("R")
_SENTINEL = object()


def once(func: Callable[..., R]) -> Callable[..., R]:
    """Ensure func is called at most once; cache and return its result."""
    result: list[R] = []   # use list to hold result (avoids nonlocal for simple assignment)

    @functools.wraps(func)
    def wrapper(*args: object, **kwargs: object) -> R:
        if not result:
            result.append(func(*args, **kwargs))
        return result[0]

    return wrapper


@once
def expensive_setup() -> dict[str, int]:
    print("Running expensive setup...")
    return {"key": 42}

print(expensive_setup())  # >>> Running expensive setup... {'key': 42}
print(expensive_setup())  # >>> {'key': 42}  — no "Running..." message
print(expensive_setup())  # >>> {'key': 42}
```

Using a `list[R]` as a container for the cached value avoids needing `nonlocal` (which would require a typed variable that can hold either a sentinel or the real result). The first call appends the result; subsequent calls find a non-empty list and return `result[0]` directly.

## Sources

- Python Function Definitions — https://docs.python.org/3/reference/compound_stmts.html#function-definitions
- PEP 3107 — Function Annotations — https://peps.python.org/pep-3107/
- PEP 570 — Python Positional-Only Parameters — https://peps.python.org/pep-0570/
- PEP 612 — Parameter Specification Variables (`ParamSpec`) — https://peps.python.org/pep-0612/
- `functools` documentation — https://docs.python.org/3/library/functools.html
- Ramalho, L. *Fluent Python* (2nd ed., 2022). Chapters 7 and 9.
- Raymond Hettinger, "Transforming Code into Beautiful, Idiomatic Python" (PyCon 2013).

## Related

- [2 - The Data Model — Objects, References, Identity](./2-the-data-model-objects-references-identity.md)
- [3 - Iterables, Iterators, and Generators](./3-iterables-iterators-and-generators.md)
- [5 - Classes, Inheritance, MRO, ABCs](./5-classes-inheritance-mro-abcs.md)
- [6 - Type Hints and Static Typing](./6-type-hints-and-static-typing.md)
- [7 - Exceptions and Context Managers](./7-exceptions-and-context-managers.md)
