---
title: "Introduction, Analysis of Algorithms, and Recursion"
created: 2026-05-19
updated: 2026-05-19
tags: [dsa, algorithms, recursion, complexity, big-o, asymptotic-notation]
aliases: ["intro recursion", "algorithm analysis", "big o notation"]
---

# Introduction, Analysis of Algorithms, and Recursion

[toc]

> **TL;DR:** An algorithm's correctness is a given — what distinguishes good from bad is *how fast it grows* as input scales. Asymptotic notation (Big-O, Theta, Omega) strips away machine-specific constants to express that growth rate mathematically. Recursion is a program structure where a function calls itself on a strictly smaller sub-problem until it hits a base case; the call stack physically embodies the deferred work, and its depth determines the space cost.

## Vocabulary

**Algorithm** — a finite, deterministic sequence of instructions that solves a class of problems. Correctness is necessary but not sufficient; efficiency is the differentiator.

**Data Structure** — a concrete storage and access layout for data (array, linked list, tree, hash table). Defines which operations are cheap and which are expensive.

**Abstract Data Type (ADT)** — a mathematical specification of a data structure by its *interface* (insert, delete, search, etc.) without specifying the implementation. The stack ADT can be implemented with an array or a linked list.

**Time Complexity** — the number of elementary operations an algorithm performs as a function of input size n. Measured in the worst, average, or best case.

**Space Complexity** — the total memory an algorithm uses as a function of n, including both the input storage and *auxiliary* (extra) space.

**Asymptotic Notation** — the family of mathematical notations (O, Theta, Omega, o, omega) that describe the *limiting behaviour* of a function as n approaches infinity, discarding constants and lower-order terms.

**Big-O — O(g(n))** — an *upper bound*: f(n) is O(g(n)) if there exist constants c > 0 and n_0 such that f(n) <= c * g(n) for all n >= n_0. "f grows no faster than g."

**Big-Omega — Omega(g(n))** — a *lower bound*: f(n) is Omega(g(n)) if f(n) >= c * g(n) for all n >= n_0. "f grows at least as fast as g."

**Big-Theta — Theta(g(n))** — an *exact bound*: f(n) is Theta(g(n)) if f(n) is both O(g(n)) and Omega(g(n)). The best characterisation; use it when you can prove both sides.

**Order of Growth** — the dominant term in a complexity expression after dropping constants. Predicts worst-case execution time and memory for large n.

**Recurrence Relation** — an equation that expresses T(n) in terms of T on a smaller argument (e.g. T(n) = T(n-1) + O(1)). Solved by substitution, recursion-tree, or Master Theorem.

**Base Case** — the condition in a recursive function that terminates recursion without a further recursive call. Without a base case, recursion is infinite.

**Recursive Case** — the branch of a recursive function that calls itself on a *strictly smaller* sub-problem, moving toward the base case.

**Stack Frame** — the activation record pushed onto the call stack when a function is invoked. Contains the function's local variables, parameters, and the return address. Each pending recursive call occupies one stack frame.

**Tail Recursion** — a special form of recursion where the recursive call is the *last* action in the function — no work happens after the call returns. Tail-recursive functions can be compiled to a loop in languages that support tail-call elimination (Scheme, Haskell, Scala; notably NOT CPython).

**Memoization** — caching the result of a recursive call keyed by its arguments so that repeated sub-problems are solved only once. Bridges naive recursion and dynamic programming (covered in detail in the DP note).

## Intuition

Think of complexity analysis as answering one question: if I double the input size, what happens to the runtime? For an O(n) algorithm it also doubles. For O(n^2) it quadruples. For O(log n) it barely moves. The constants — which processor, which language, which cache — matter for microsecond-level profiling, not for architectural decisions. Asymptotic notation formalises that "ignore the constants" intuition.

Recursion is the code-level expression of *mathematical induction*. You trust that `solve(n-1)` already returns the right answer, and you only have to figure out how to combine it with the current step to get `solve(n)`. The call stack is the machine's way of remembering all the "not yet combined" partial results.

> [!IMPORTANT]
> Order of growth gives the *worst-case possibility*. Two algorithms that are both O(n^2) in the worst case can differ by 100x on real inputs because of constants, cache behaviour, or average-case characteristics. Never stop the analysis at the Big-O class.

## Math foundations

The algorithms and complexity arguments throughout this note rest on five mathematical pillars. If any of them feel shaky, this section is the refresher. Each sub-section is self-contained: read it, work the example, move on.

> [!IMPORTANT]
> Mathematical induction is the proof technique that *justifies* recursion. Every correct recursive function is, implicitly, an inductive argument: the base case establishes the claim for the smallest input, and the recursive case shows that if it holds for n-1 it holds for n. Understanding induction makes recursive correctness proofs mechanical.

### Mathematical induction

Induction proves a statement P(n) holds for all natural numbers n >= n_0 by two steps: the *base case* establishes P(n_0) directly, and the *inductive step* shows that P(k) implies P(k+1) for any k >= n_0. This is structurally identical to a recursive function: the base case is the terminating branch, and the inductive step is the recursive branch that assumes the sub-problem is already solved correctly.

**Worked example — prove that 1 + 2 + ... + n = n(n+1)/2.**

```math
\text{Claim: } S(n) = \sum_{i=1}^{n} i = \frac{n(n+1)}{2}
```

*Base case (n = 1):* S(1) = 1, and 1(1+1)/2 = 1. Check.

*Inductive step:* Assume S(k) = k(k+1)/2 for some k >= 1. Show S(k+1) = (k+1)(k+2)/2.

```math
S(k+1) = S(k) + (k+1) = \frac{k(k+1)}{2} + (k+1) = (k+1)\left(\frac{k}{2} + 1\right) = \frac{(k+1)(k+2)}{2}
```

The structural parallel to code is exact: `recursive_sum(n)` trusts `recursive_sum(n-1)` returns the right answer (inductive hypothesis), then adds n to get `recursive_sum(n)` (inductive step).

### Recurrence relations

A recurrence relation expresses T(n) — the cost of solving a problem of size n — in terms of T on smaller arguments. Solving a recurrence means finding a closed form (a non-recursive formula in terms of n alone). The standard technique is *back-substitution*: expand the right-hand side repeatedly, recognise a pattern, and evaluate at the base case.

**Case 1 — T(n) = T(n-1) + c (linear recursion: factorial, sum, Josephus).**

Substitute repeatedly: T(n) = T(n-2) + c + c = T(n-k) + kc. At k = n-1 we hit T(1) = c:

```math
T(n) = T(1) + (n-1)c = c + (n-1)c = nc = \Theta(n)
```

**Case 2 — T(n) = 2T(n/2) + cn (divide-and-conquer with linear merge: merge sort).**

This is Master Theorem Case 2: a = 2, b = 2, f(n) = cn, and n^(log_b a) = n^(log_2 2) = n^1 = n. Since f(n) = Theta(n) = Theta(n^(log_b a)), the solution is:

```math
T(n) = \Theta\!\left(n \log n\right)
```

The recursion-tree argument: there are log_2(n) levels; every level costs cn total (the merge step distributes the n elements across all sub-problems at that level). Multiply: cn * log_2(n) = Theta(n log n).

> [!TIP]
> The Master Theorem gives closed forms instantly for T(n) = aT(n/b) + f(n). Memorise the three cases by which term dominates: leaves dominate (Case 1, T = Theta(n^(log_b a))), tied (Case 2, T = Theta(n^(log_b a) log n)), root dominates (Case 3, T = Theta(f(n))). For T(n) = 2T(n/2) + n, the merge work ties the leaves — Case 2.

### Logarithms — why log n is everywhere

Whenever an algorithm repeatedly halves its input — binary search, merge sort's split phase, a balanced BST traversal — the question "how many times can I halve n before reaching 1?" has the answer log_2(n). That is the definition of the logarithm: log_2(n) is the exponent k such that 2^k = n.

In Big-O, the base of the logarithm is irrelevant. The change-of-base formula shows why: log_a(n) = log_b(n) / log_b(a). The denominator log_b(a) is a constant, absorbed into the Big-O constant. So O(log_2 n) = O(log_3 n) = O(log n) — we always write log n without a base in asymptotic notation.

Three identities that appear constantly in algorithm proofs:

```math
\log(ab) = \log a + \log b
```

```math
\log\!\left(\frac{a}{b}\right) = \log a - \log b
```

```math
\log(a^b) = b \log a
```

The third identity is why the depth of a complete binary tree of n nodes is log_2(n): solving 2^k = n gives k = log_2(n).

### Summations and geometric series

Loop analyses and recursion-tree level-cost calculations reduce to summation closed forms. Three identities are load-bearing for DSA:

**Arithmetic series** (used for quadratic-loop analysis, triangular numbers):

```math
\sum_{i=1}^{n} i = 1 + 2 + \cdots + n = \frac{n(n+1)}{2} = \Theta(n^2)
```

**Sum of squares** (appears in some graph and matrix algorithms):

```math
\sum_{i=1}^{n} i^2 = \frac{n(n+1)(2n+1)}{6} = \Theta(n^3)
```

**Geometric series** (used for exponential-recursion analysis; e.g., T(n) = 2T(n-1) + c gives a sum of powers of 2):

```math
\sum_{i=0}^{k} r^i = 1 + r + r^2 + \cdots + r^k = \frac{r^{k+1} - 1}{r - 1} \quad (r \neq 1)
```

For r = 2 and k = n-1: the sum is 2^n - 1. This is exactly the move count in Tower of Hanoi and the node count in a complete binary tree of height n.

**Why these appear in recursion-tree analysis:** each level of the recursion tree contributes some cost. When the sub-problem count doubles but sub-problem size halves (merge sort), the level costs are constant — arithmetic series. When the sub-problem count doubles and sub-problem size *also* shrinks to 1 (Tower of Hanoi), level costs form a geometric series with ratio 2.

### Limits and rates of growth — the formal Big-O definition

Big-O is not "approximately" or "roughly" — it is a precise mathematical statement about the *eventual* dominance of one function over another. The formal definition: f(n) = O(g(n)) if and only if there exist constants c > 0 and n_0 >= 0 such that f(n) <= c * g(n) for all n >= n_0. Omega and Theta are symmetric:

```math
f(n) = O(g(n)) \iff \exists\, c > 0,\, n_0 \geq 0 \text{ s.t. } f(n) \leq c \cdot g(n)\ \forall\, n \geq n_0
```

```math
f(n) = \Omega(g(n)) \iff \exists\, c > 0,\, n_0 \geq 0 \text{ s.t. } f(n) \geq c \cdot g(n)\ \forall\, n \geq n_0
```

```math
f(n) = \Theta(g(n)) \iff f(n) = O(g(n)) \text{ and } f(n) = \Omega(g(n))
```

**Worked example — prove 3n^2 + 5n = O(n^2).**

We need c and n_0 such that 3n^2 + 5n <= c * n^2 for all n >= n_0. Divide both sides by n^2 (valid for n > 0): 3 + 5/n <= c. As n grows, 5/n shrinks toward 0, so for any n >= 1 we have 3 + 5/n <= 3 + 5 = 8. Take c = 8, n_0 = 1:

```math
3n^2 + 5n \leq 8n^2 \quad \forall\, n \geq 1
```

Done. The 5n term is "swallowed" by the extra 5n^2 slack between 3n^2 and 8n^2.

> [!NOTE]
> The limit definition is often faster for informal proofs: if lim(n -> infinity) f(n)/g(n) = L where 0 < L < infinity, then f(n) = Theta(g(n)). If L = 0, f = o(g) (strictly dominated). For 3n^2 + 5n versus n^2: the limit is (3n^2 + 5n)/n^2 = 3 + 5/n -> 3, a positive finite constant, confirming Theta(n^2).


## How it works

### Three algorithms for the same problem

The notes open with a canonical example: compute the sum of the first n natural numbers. Three algorithms with wildly different complexities solve *exactly the same problem*, which makes the comparison unambiguous.

```python
def fun1(n: int) -> int:
    """O(1) — closed-form formula. No loop, no recursion."""
    return n * (n + 1) // 2

def fun2(n: int) -> int:
    """O(n) — single loop, one addition per iteration."""
    total: int = 0
    for i in range(1, n + 1):
        total += i
    return total

def fun3(n: int) -> int:
    """O(n^2) — nested loop, inner loop runs i times for each i."""
    total: int = 0
    for i in range(1, n + 1):
        for j in range(1, i + 1):
            total += 1
    return total
```

The cost of `fun1` is fixed — two arithmetic ops and a division, regardless of n. The cost of `fun2` scales linearly: n iterations, one addition each. The cost of `fun3` is quadratic: the inner loop runs 1 + 2 + ... + n = n(n+1)/2 times total. In terms of the notes' notation: `fun1() → c1`, `fun2() → c1 + c2*n`, `fun3() → c1 + c2*n + c3*n^2`.

### Asymptotic notation — the formal definitions

Asymptotic notation exists to compare algorithms in a machine-independent way. The notes motivate this concretely: if algorithm 1 runs on a slow machine and slow language, the constant c can come out to 1000. But as n grows toward infinity, the *order of growth* overwhelms that constant — there will always be an n beyond which O(n) beats O(n^2), no matter how large the constant.

The three notations form a sandwich. For a function f(n) = 2n^2 + 5:

- **Upper bound** (Big-O): f(n) is O(n^2) because 2n^2 + 5 <= 3n^2 for all n >= 5 (take c = 3, n_0 = 5). Also O(n^3) — it just isn't tight.
- **Lower bound** (Omega): f(n) is Omega(n^2) because 2n^2 + 5 >= 2n^2 for all n >= 0 (take c = 2, n_0 = 0).
- **Tight bound** (Theta): f(n) is Theta(n^2) because both bounds hold.

> [!NOTE]
> The limit definition is often cleaner to apply: if lim(n→∞) f(n)/g(n) equals a positive finite constant, then f(n) is Theta(g(n)). If the limit is 0, then f is o(g) (strictly dominated). If the limit is infinity, f is omega(g) (strictly dominates).

The key comparison result from the notes — the order of growth hierarchy (slowest to fastest):

```
1  <  log(log n)  <  log n  <  n^(1/2)  <  n  <  n^2  <  n^3  <  2^n  <  n!
```

### Big-O notation — worked proof

To prove `2n + 3 = O(n)`, we need to find c > 0 and n_0 such that 2n + 3 <= c * n for all n >= n_0.

Take c = 3. Then 2n + 3 <= 3n requires n >= 3. So n_0 = 3, c = 3, and the definition is satisfied.

The notes show this graphically: for n >= n_0, the curve c*g(n) lies above f(n), and Big-O is the upper-bounding curve.

> [!TIP]
> When proving Big-O, choose c to be the sum of the leading coefficient and 1 (or any convenient larger value), then solve for the crossover point n_0. The choice of c and n_0 is not unique — any valid pair works.

### Omega notation — lower bound

Omega is the opposite direction: f(n) = Omega(g(n)) means g(n) is a lower bound on f(n). For `2n + 3 = Omega(n)`: take c = 2, n_0 = 0. Then 2n + 3 >= 2n for all n >= 0. Done.

In algorithm analysis, Omega gives *best-case lower bounds*. Saying comparison-based sorting is Omega(n log n) means no comparison sort can do better — it is an *inherent* lower bound on the problem, not just a specific algorithm.

### Theta notation — tight bound

Theta pins f(n) between two multiples of g(n): c1 * g(n) <= f(n) <= c2 * g(n) for all n >= n_0. The notes' example: `f(n) = 2n + 3`, `g(n) = n`. Take c1 = 2, c2 = 3, n_0 = 3. Then 2n <= 2n + 3 <= 3n for all n >= 3. Hence `2n + 3 = Theta(n)`.

Theta is the most informative notation. When you can prove it, use it. Big-O alone is technically correct but weak — "O(n!)" is true of every polynomial algorithm, which tells you nothing useful.

### Best case, average case, worst case

These are *input-model* choices, not notation choices. For linear search on an unsorted array of size n:

- **Best case**: the target is at index 0. One comparison. Theta(1).
- **Average case**: the target is equally likely at any position. Expected n/2 comparisons. Theta(n).
- **Worst case**: the target is at index n-1 or absent. n comparisons. Theta(n).

The notes give linear search as the worked example. The average case assumes uniform distribution; in practice the distribution matters and the average case can be hard to characterise exactly.

### Analysis of common loops

The notes systematically derive the complexity of common loop structures. Each pattern maps directly to a standard growth class:

**Pattern 1 — linear (`for i in range(n)`)**: body runs n times. Theta(n).

**Pattern 2 — logarithmic (halving loop `while i < n: i *= c`)**: i takes values c^0, c^1, c^2, ..., c^k until c^k >= n. So k = log_c(n). Theta(log n).

**Pattern 3 — quadratic nested loop `for i ... for j in range(i)`)**: inner runs 0+1+2+...+(n-1) = n(n-1)/2 times. Theta(n^2).

**Pattern 4 — n log n (outer linear, inner halving)**: outer runs n times; inner runs log n times each. Total: n * log n. Theta(n log n).

```python
def loop_examples(n: int) -> None:
    # Theta(n) — linear
    i = 1
    while i <= n:
        i += 1          # i increments by 1 each time

    # Theta(log n) — logarithmic
    i = 1
    while i <= n:
        i *= 2          # i doubles each time: 1, 2, 4, 8, ...

    # Theta(log n) — logarithmic with base c
    c = 3
    i = 1
    while i <= n:
        i *= c          # generalised: log_c(n) iterations

    # Theta(sqrt(n)) — square-root loop (less common)
    i = 1
    while i * i <= n:
        i += 1
```

### Recursion fundamentals

A recursive function solves a problem by reducing it to one or more *smaller instances of the same problem*. Every correct recursive function has two structural requirements: a base case that terminates the recursion, and a recursive case that makes *strict progress* toward the base case (the argument must shrink — by subtraction, halving, or another measure).

The notes use the sum-of-natural-numbers function as the first recursion example:

```python
def recursive_sum(n: int) -> int:
    """Sum 1 + 2 + ... + n via recursion. T(n) = T(n-1) + O(1) => O(n)."""
    if n == 0:          # base case
        return 0
    return n + recursive_sum(n - 1)   # recursive case
```

> [!IMPORTANT]
> The recursive case must make the argument *strictly smaller*. `recursive_sum(n)` calls `recursive_sum(n-1)` — n decreases by 1 each step. If you accidentally call `recursive_sum(n)` from inside itself (no progress), you get infinite recursion and a stack overflow.

### The call stack — frame-by-frame for factorial(4)

Each call to a function pushes a stack frame. The frame stores the local variables and the return address. For recursive functions, frames accumulate until the base case is hit, then they unwind in reverse order.

**Figure:** call stack growth and unwind for `factorial(4)`.

```
CALL PHASE (stack grows downward)
─────────────────────────────────────────────────────────
factorial(4)  → waiting for factorial(3)
  ├─ frame: n=4, returns 4 * <pending>
  │
  factorial(3)  → waiting for factorial(2)
    ├─ frame: n=3, returns 3 * <pending>
    │
    factorial(2)  → waiting for factorial(1)
      ├─ frame: n=2, returns 2 * <pending>
      │
      factorial(1)  → waiting for factorial(0)
        ├─ frame: n=1, returns 1 * <pending>
        │
        factorial(0)  ← BASE CASE REACHED
          └─ frame: n=0, returns 1

RETURN PHASE (stack unwinds)
─────────────────────────────────────────────────────────
factorial(0) returns 1
  → factorial(1) gets 1, returns 1 * 1 = 1
    → factorial(2) gets 1, returns 2 * 1 = 2
      → factorial(3) gets 2, returns 3 * 2 = 6
        → factorial(4) gets 6, returns 4 * 6 = 24
```

Stack depth is n+1 frames. If n = 100,000, that is 100,001 frames on the stack — this is why Python raises `RecursionError` at depth ~1000 by default.

```python
def factorial(n: int) -> int:
    """n! via recursion. T(n) = T(n-1) + O(1) => O(n) time, O(n) space."""
    if n == 0:      # base case: 0! = 1
        return 1
    return n * factorial(n - 1)
```

### Fibonacci — naive recursion and its catastrophic cost

The Fibonacci sequence is the canonical example of *exponential blowup* from naive recursion. The notes show the recursion tree explicitly: `fib(5)` spawns `fib(4)` and `fib(3)`; `fib(4)` spawns `fib(3)` and `fib(2)` — `fib(3)` is computed *twice*, `fib(2)` three times, `fib(1)` five times. The number of calls doubles at each level.

**Figure:** recursion tree for `fib(5)` — repeated sub-problems visible.

```
                   fib(5)
                 /        \
           fib(4)          fib(3)
          /      \         /    \
       fib(3)  fib(2)  fib(2)  fib(1)
       /   \   /   \   /   \
   fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
   /   \
fib(1) fib(0)

Nodes marked with same label are redundant computations.
Total calls: O(2^n)
```

```python
def fib_naive(n: int) -> int:
    """Naive Fibonacci. T(n) = 2T(n-1) + O(1) => O(2^n). Do not use for n > 40."""
    if n == 0 or n == 1:   # base cases: fib(0)=0, fib(1)=1
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)


def fib_memo(n: int, memo: dict[int, int] | None = None) -> int:
    """Memoized Fibonacci. O(n) time, O(n) space. Each sub-problem solved once."""
    if memo is None:
        memo = {}
    if n in memo:
        return memo[n]
    if n == 0 or n == 1:
        return n
    memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]
```

The memoized version reduces the recursion tree to a chain — each of fib(0) through fib(n) is computed exactly once.

### Tower of Hanoi — recursion on a non-obvious structure

Tower of Hanoi is the standard example where the recursive structure is elegant but not immediately obvious from the problem statement. You have n disks on peg A and must move all to peg C, using peg B as auxiliary, with the constraint that a larger disk can never rest on a smaller one.

The recursive insight: to move n disks from A to C using B, you (1) move n-1 disks from A to B using C, (2) move the largest disk from A to C directly, (3) move n-1 disks from B to C using A. Steps 1 and 3 are recursive sub-problems of size n-1. Step 2 is the base-case move (when n=1, just move the one disk).

**Figure:** three-disk Hanoi decomposition.

```
n=3: move 3 disks A→C via B
├─ n=2: move 2 disks A→B via C
│   ├─ n=1: move disk from A→C (move 1)
│   ├─ move disk from A→B (move 2)
│   └─ n=1: move disk from C→B (move 3)
├─ move largest disk from A→C (move 4)
└─ n=2: move 2 disks B→C via A
    ├─ n=1: move disk from B→A (move 5)
    ├─ move disk from B→C (move 6)
    └─ n=1: move disk from A→C (move 7)

Total moves = 2^n - 1 = 7 for n=3
```

```python
def tower_of_hanoi(n: int, source: str, auxiliary: str, destination: str) -> int:
    """
    Move n disks from source to destination using auxiliary.
    Returns the total number of moves (always 2^n - 1).
    T(n) = 2T(n-1) + 1  =>  O(2^n)
    """
    if n == 1:   # base case
        print(f"Move disk 1 from {source} to {destination}")
        return 1
    moves: int = 0
    moves += tower_of_hanoi(n - 1, source, destination, auxiliary)
    print(f"Move disk {n} from {source} to {destination}")
    moves += 1
    moves += tower_of_hanoi(n - 1, auxiliary, source, destination)
    return moves
```

### Tail recursion

A function is tail-recursive when the recursive call is the *very last* thing executed — the caller does no computation after the recursive call returns. This matters because a tail call does not need to preserve its own stack frame: the callee can reuse the caller's frame. Languages that implement tail-call optimisation (TCO) convert tail-recursive functions to loops at compile time, giving O(1) stack space instead of O(n).

The notes distinguish the two forms clearly:

```python
# NON-tail-recursive: multiplication happens AFTER recursive_sum returns
def recursive_sum(n: int) -> int:
    if n == 0:
        return 0
    return n + recursive_sum(n - 1)   # n + (...) — work pending after return

# Tail-recursive: accumulator carries the partial sum, no work after the call
def recursive_sum_tail(n: int, accumulator: int = 0) -> int:
    if n == 0:
        return accumulator             # just return; nothing pending
    return recursive_sum_tail(n - 1, accumulator + n)   # tail call
```

> [!WARNING]
> CPython does NOT implement tail-call optimisation. `recursive_sum_tail(1_000_000)` will still raise `RecursionError`. The tail-recursive form is pedagogically cleaner and translates naturally to a loop, but in Python you must manually write the loop to get the O(1) stack benefit.

### Subsets, permutations, and the Josephus problem

The notes cover three more classical recursion problems. These are important because they illustrate different recursion patterns beyond the simple linear chain.

**Generate all subsets** of a string — at each character, you either include it or exclude it. The decision tree has depth equal to string length, with 2^n leaves.

**Permutations** — at each position, fix one of the remaining characters and recurse on the rest. The decision tree has n! leaves.

**Josephus problem** — n people standing in a circle; every k-th person is eliminated; find the survivor's position. The elegant recursion: `jos(n, k) = (jos(n-1, k) + k) % n`, base case `jos(1, k) = 0`. This is a recurrence that reduces an n-person problem to an (n-1)-person problem. Time complexity: T(n) = T(n-1) + O(1) = O(n).

```python
def josephus(n: int, k: int) -> int:
    """
    Return the 0-indexed position of the survivor among n people,
    eliminating every k-th person.
    T(n) = T(n-1) + O(1) => O(n)
    """
    if n == 1:
        return 0
    return (josephus(n - 1, k) + k) % n
```

## Math

### Recurrence relations and their solutions

The notes derive complexities by solving recurrences. The two core techniques are *substitution (back-substitution)* and the *recursion-tree method*.

**Back-substitution for T(n) = T(n-1) + c:**

```math
T(n) = T(n-1) + c
     = T(n-2) + c + c
     = T(n-3) + 3c
     = \ldots
     = T(1) + (n-1)c
     = \Theta(n)
```

This is the recurrence for linear recursion (factorial, sum, Josephus).

**Back-substitution for T(n) = 2T(n-1) + c (Tower of Hanoi, naive Fibonacci):**

```math
T(n) = 2T(n-1) + c
     = 2(2T(n-2) + c) + c = 4T(n-2) + 2c + c
     = 2^k T(n-k) + c(2^k - 1)
```

At k = n-1, T(1) = c, so:

```math
T(n) = 2^{n-1} \cdot c + c(2^{n-1} - 1) = c \cdot 2^n - c = \Theta(2^n)
```

**Back-substitution for T(n) = T(n/2) + c (binary search, halving loops):**

```math
T(n) = T(n/2) + c
     = T(n/4) + c + c
     = T(n/2^k) + kc
```

At k = log_2(n), T(1) = c:

```math
T(n) = c + c \log_2 n = \Theta(\log n)
```

**Back-substitution for T(n) = 2T(n/2) + cn (merge sort pattern):**

```math
T(n) = 2T(n/2) + cn
```

Recursion tree has log_2(n) levels; each level costs cn total work (the merge step). Total: cn * log_2(n).

```math
T(n) = \Theta(n \log n)
```

### Master Theorem (preview)

The notes introduce the recursion-tree method as a path toward the Master Theorem. The Master Theorem is the closed-form solution for recurrences of the form:

```math
T(n) = aT(n/b) + f(n)
```

where a >= 1 (branching factor), b > 1 (sub-problem size reduction), and f(n) is the cost of work done outside the recursive calls. Three cases based on the comparison between f(n) and n^(log_b(a)):

- **Case 1**: f(n) is polynomially smaller than n^(log_b a) → T(n) = Theta(n^(log_b a))
- **Case 2**: f(n) = Theta(n^(log_b a)) → T(n) = Theta(n^(log_b a) * log n)
- **Case 3**: f(n) is polynomially larger than n^(log_b a) → T(n) = Theta(f(n))

### Space complexity — auxiliary vs. total

Total space = input storage + auxiliary storage. For most analyses we care about auxiliary space (extra memory beyond the input).

Recursive sum: O(n) auxiliary (n stack frames, each O(1) in size).
Iterative sum: O(1) auxiliary (one accumulator variable).
Fibonacci naive: O(n) auxiliary (max stack depth is n).
Fibonacci memoized: O(n) auxiliary (the memo dictionary holds at most n entries).

```math
\text{Space(recursive factorial)} = O(n)   \quad \text{(one frame per call)}
\text{Space(iterative factorial)} = O(1)   \quad \text{(one loop variable)}
```

### Big-O growth rate table

| Complexity | Name | n = 10 | n = 100 | n = 1000 | n = 10^6 |
| :--- | :--- | ---: | ---: | ---: | ---: |
| O(1) | constant | 1 | 1 | 1 | 1 |
| O(log n) | logarithmic | 3 | 7 | 10 | 20 |
| O(n) | linear | 10 | 100 | 1,000 | 1,000,000 |
| O(n log n) | linearithmic | 33 | 665 | 9,966 | ~20M |
| O(n^2) | quadratic | 100 | 10,000 | 1,000,000 | 10^12 |
| O(2^n) | exponential | 1,024 | ~10^30 | infeasible | infeasible |
| O(n!) | factorial | 3.6M | ~10^157 | infeasible | infeasible |

Numbers are operation counts (approximate). Assumes 10^9 operations/sec: an O(2^n) algorithm is fundamentally unusable past n ~ 60.

## Real-world example

### Directory tree traversal

The Unix filesystem is a recursive structure: a directory contains files and subdirectories, each subdirectory contains its own files and subdirectories, and so on. Traversal is naturally recursive — walk the current directory, then recurse into each subdirectory. This is a real production operation: `du`, `find`, `rsync`, and most build systems implement this pattern.

The recursion here is over the *depth* of the directory tree, not a numeric n. Each directory is a sub-problem of the same shape. The base cases are: (1) an empty directory — return 0 bytes, and (2) a regular file — return its size.

```python
import os
from pathlib import Path


def directory_size(path: Path) -> int:
    """
    Recursively compute the total byte size of a directory tree.

    Time:  O(N)  where N = total number of files and directories.
    Space: O(D)  where D = maximum depth of the directory tree
                 (one stack frame per level of depth).

    Real-world analogues: 'du -sh', rsync file enumeration, build system
    dependency scanning.
    """
    if path.is_file():                    # base case 1: a regular file
        return path.stat().st_size

    if not path.is_dir():                 # base case 2: symlink, device, etc.
        return 0

    total: int = 0
    for child in path.iterdir():          # recursive case: iterate children
        total += directory_size(child)    # each child is a sub-problem

    return total


def directory_tree(path: Path, indent: int = 0) -> None:
    """
    Print a tree view of a directory (like 'tree' command).
    Shows the recursion structure visually.
    """
    prefix = "  " * indent
    print(f"{prefix}{path.name}/")
    if path.is_dir():
        for child in sorted(path.iterdir()):
            if child.is_dir():
                directory_tree(child, indent + 1)
            else:
                print(f"{prefix}  {child.name}  ({child.stat().st_size} bytes)")
```

> [!TIP]
> In production, replace `path.iterdir()` with `os.scandir()` — it returns `DirEntry` objects that cache the `is_file()` / `is_dir()` result, avoiding a second `stat()` syscall per entry. For very deep trees (virtualenv inside a node_modules inside a docker layer), the O(D) stack depth can still bite you; `os.walk()` uses an explicit stack internally and avoids this.

> [!WARNING]
> Symlinks can create cycles in the directory graph. Production implementations must track visited inodes (`os.stat().st_ino`) to avoid infinite recursion on circular symlinks.

## In practice

### Python's recursion limit is your enemy

CPython has a default call stack depth limit of 1000 (retrievable and settable via `sys.getrecursionlimit()` / `sys.setrecursionlimit()`). This is a hard cap enforced in the C interpreter loop — not a soft hint. Hitting it raises `RecursionError`, which is not a warning, it crashes the call.

```python
import sys

print(sys.getrecursionlimit())   # 1000 on stock CPython

# Temporarily raise it for a deep recursive algorithm:
sys.setrecursionlimit(10_000)
# Danger: you're now relying on the OS thread stack not overflowing.
# Each Python frame is ~1–2 KB; 10,000 frames = 10–20 MB of stack.
# Default thread stack size on Linux/macOS is 8 MB.
```

> [!CAUTION]
> Raising `sys.setrecursionlimit` past ~10,000 risks a C-level stack overflow that kills the process with a segfault — no Python exception, no clean exit. The correct fix is to convert deep recursion to an explicit stack (iterative DFS) or to use memoization/DP to avoid deep call chains entirely.

### When iteration beats recursion in Python

For numeric recursion (factorial, Fibonacci, sum), the iterative version is always faster in CPython because:

1. Each function call has overhead: creating a frame object, pushing it to the call stack, looking up the return address.
2. The recursive version has O(n) space cost (frames); the iterative version has O(1).
3. CPython does not optimise tail calls — the transformation is on you.

```python
def factorial_iterative(n: int) -> int:
    """O(n) time, O(1) space. Use this in production; never the recursive version."""
    result: int = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

Rule of thumb: if the algorithm has a natural iterative formulation and the recursion depth can be large, prefer the loop. Reserve recursion for structures that are inherently recursive (trees, graphs, divide-and-conquer) where the depth is bounded by the tree height (typically O(log n) for balanced trees).

### When recursion is the right tool

Recursive code is shorter, often provably correct by structural induction, and maps directly to the problem's mathematical definition. Use it when:

- The data structure is inherently recursive (binary tree, JSON, AST, filesystem hierarchy).
- The problem decomposes cleanly into sub-problems of the same shape (divide-and-conquer: merge sort, quicksort, binary search).
- The recursion depth is provably bounded and small (e.g. O(log n) for balanced BST operations with n <= 10^7 gives depth <= 23 — well within the stack limit).

### Memoization vs. tabulation

Memoized recursion (top-down DP) and tabulated iteration (bottom-up DP) solve the same problem. Memoization is easier to implement correctly — you write the recursive formulation, add a cache, done. Tabulation fills a table in dependency order and avoids stack overhead entirely. For problems with large n or where stack depth is a concern, tabulation is the production choice.

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_cached(n: int) -> int:
    """lru_cache is Python's built-in memoization decorator. O(n) time, O(n) space."""
    if n < 2:
        return n
    return fib_cached(n - 1) + fib_cached(n - 2)
```

## Pitfalls

- **"Big-O tells me which algorithm is faster."** Big-O tells you the *asymptotic growth rate*. For small n, a higher-complexity algorithm with a small constant can be faster than a lower-complexity one with a large constant. Measure both when in doubt.

- **"O(n) is always better than O(n^2)."** For n = 10, an O(n^2) algorithm doing 100 elementary ops beats an O(n) algorithm doing 1000 ops. Cross-over points matter in practice. The statement becomes true as n → ∞.

- **"I have a base case, so my recursion terminates."** A base case is necessary but not sufficient. The recursive call must make progress toward the base case on every path. `def f(n): return f(n-1) if n > 0 else 0` terminates, but `def f(n): return f(n+1) if n > 0 else 0` does not, despite having a base case.

- **"Tail recursion is optimised in Python."** It is not. CPython has deliberately not implemented TCO. Guido van Rossum's stated reason: Python's tracebacks need the full call stack for debugging. A tail-recursive function in Python still consumes O(n) stack frames.

- **"Memoization makes the algorithm O(1) space."** Memoization trades time for space — it stores all previously computed results. A memoized Fibonacci with input n stores O(n) results. The space cost moved from the call stack to the heap, but the total is still O(n).

- **"Omega means best-case."** Omega is a lower bound on the *function's growth*, not necessarily the best-case input. Saying merge sort is Omega(n log n) means no comparison sort can do better *in the worst case* — it is a statement about the problem's inherent difficulty, not about lucky inputs.

- **"Average case is just the arithmetic mean of all cases."** Average-case complexity requires specifying a probability distribution over inputs. The arithmetic mean over *uniformly random* inputs is one model; the distribution you actually see in production may be very different.

## Exercises

### Exercise 1 — Recursive sum of list elements

Given a list of integers, return their sum using recursion. Do not use Python's built-in `sum()`.

> [!TIP]
> Base case: an empty list has sum 0. Recursive case: the sum of a list is its first element plus the sum of the rest.

**Solution:**

```python
def list_sum(arr: list[int]) -> int:
    """
    Recursive sum of a list.
    T(n) = T(n-1) + O(1) => O(n) time, O(n) space (stack depth = len(arr)).
    """
    if len(arr) == 0:          # base case: empty list
        return 0
    return arr[0] + list_sum(arr[1:])   # head + sum(tail)
```

**Complexity:** Time O(n), Space O(n) — both because we make n recursive calls. Note that `arr[1:]` creates a new list each call, adding another O(n) allocation cost; passing an index avoids this.

```python
def list_sum_idx(arr: list[int], i: int = 0) -> int:
    """Same logic, O(1) allocation per call by passing index instead of slicing."""
    if i == len(arr):
        return 0
    return arr[i] + list_sum_idx(arr, i + 1)
```

### Exercise 2 — Reverse a string recursively

Given a string s, return its reverse using recursion.

> [!TIP]
> The reverse of a string is its last character followed by the reverse of everything except the last character. Alternatively: last_char + reverse(s[:-1]).

**Solution:**

```python
def reverse_string(s: str) -> str:
    """
    Reverse a string recursively.
    T(n) = T(n-1) + O(1) => O(n) time, O(n) space.
    String slicing in Python is O(k) for k characters — this hides a log factor
    if you care about exact constants; for learning purposes, treat each slice as O(1).
    """
    if len(s) <= 1:           # base case: empty string or single character
        return s
    return reverse_string(s[1:]) + s[0]   # reverse the tail, then append head
```

**Complexity:** T(n) = T(n-1) + O(n) (due to string concatenation being O(n) in Python). True complexity is O(n^2). For O(n), convert to a list, reverse in place, rejoin.

### Exercise 3 — Power x^n in O(log n)

Compute x^n using fast exponentiation (exponentiation by squaring). The naive approach multiplies x by itself n times — O(n). The clever approach halves the problem at each step.

> [!TIP]
> If n is even: x^n = (x^(n/2))^2. If n is odd: x^n = x * x^(n-1). This halves n each step, giving O(log n).

**Solution:**

```python
def power(x: float, n: int) -> float:
    """
    Fast exponentiation. x^n in O(log n) time, O(log n) space (stack depth = log n).
    Handles negative exponents via 1/x^(-n).
    """
    if n == 0:          # base case: anything^0 = 1
        return 1.0
    if n < 0:           # handle negative exponents
        return 1.0 / power(x, -n)
    if n % 2 == 0:      # even: x^n = (x^(n/2))^2
        half: float = power(x, n // 2)
        return half * half
    else:               # odd: x^n = x * x^(n-1)
        return x * power(x, n - 1)
```

**Complexity:** T(n) = T(n/2) + O(1) => O(log n) time, O(log n) space. Compare to the naive O(n) loop: for n = 10^9, this does ~30 multiplications vs. 10^9.

### Exercise 4 — Check if a string is a palindrome

A string is a palindrome if it reads the same forwards and backwards. Implement the check recursively.

> [!TIP]
> A string of length 0 or 1 is trivially a palindrome. For longer strings: it's a palindrome iff the first and last characters match AND the middle substring is a palindrome.

**Solution:**

```python
def is_palindrome(s: str, start: int = 0, end: int | None = None) -> bool:
    """
    Recursive palindrome check. O(n) time, O(n) space (recursion depth = n/2).
    Uses index parameters to avoid creating new strings each call.
    """
    if end is None:
        end = len(s) - 1
    if start >= end:        # base case: crossed or met in the middle
        return True
    if s[start] != s[end]:  # mismatch found
        return False
    return is_palindrome(s, start + 1, end - 1)   # check inner substring
```

**Complexity:** O(n) time (at most n/2 recursive calls), O(n) space (stack depth). An iterative two-pointer solution achieves O(1) space.

### Exercise 5 — Tower of Hanoi

Move n disks from peg A to peg C using peg B as auxiliary. Print every move.

> [!TIP]
> Think recursively: to move n disks from A to C, first move n-1 disks from A to B (out of the way), then move the largest disk directly from A to C, then move the n-1 disks from B to C on top of it.

**Solution:**

```python
def hanoi(n: int, source: str = "A", auxiliary: str = "B", dest: str = "C") -> int:
    """
    Tower of Hanoi. Prints each move and returns the total move count.
    T(n) = 2T(n-1) + 1  =>  T(n) = 2^n - 1  =>  O(2^n).
    This is optimal — 2^n - 1 moves is the *minimum* possible for n disks.
    """
    if n == 1:
        print(f"Move disk 1: {source} -> {dest}")
        return 1
    count: int = 0
    count += hanoi(n - 1, source, dest, auxiliary)   # step 1: n-1 disks A→B
    print(f"Move disk {n}: {source} -> {dest}")      # step 2: largest disk
    count += 1
    count += hanoi(n - 1, auxiliary, source, dest)   # step 3: n-1 disks B→C
    return count
```

**Complexity:** T(n) = 2T(n-1) + 1 = 2^n - 1 = O(2^n). Space O(n) — recursion depth is n. The exponential move count is inherent to the problem, not a flaw in the algorithm.

### Exercise 6 — Generate all subsets of a string

Given a string of distinct characters, print all 2^n subsets (including the empty subset).

> [!TIP]
> At each character position, you make a binary choice: include the character in the current subset or exclude it. Recurse on the remaining characters for both choices. The base case is when you've processed all characters — print the accumulated subset.

**Solution:**

```python
def all_subsets(s: str, index: int = 0, current: str = "") -> None:
    """
    Generate all 2^n subsets of string s.
    T(n) = 2T(n-1) + O(1) => O(2^n).
    Space O(n) for the recursion stack; output is 2^n strings.

    For s = "ABC": outputs "", "C", "B", "BC", "A", "AC", "AB", "ABC"
    (order depends on traversal direction).
    """
    if index == len(s):           # base case: processed all characters
        print(f'"{current}"')
        return
    # Exclude current character
    all_subsets(s, index + 1, current)
    # Include current character
    all_subsets(s, index + 1, current + s[index])
```

**Complexity:** O(2^n) time — the recursion tree has 2^n leaves. Space O(n) for the stack. The output itself is 2^n strings of average length n/2, so printing takes O(n * 2^n) total.

### Exercise 7 — Generate all permutations of a string

Given a string, print all n! permutations.

> [!TIP]
> Fix one character at each position and recurse on the remaining characters. At position 0, you can place any of the n characters; after placing one, recurse to fill positions 1 through n-1 with the remaining n-1 characters.

**Solution:**

```python
def permutations(s: str, start: int = 0) -> None:
    """
    Generate all n! permutations of s by swapping in-place.
    Operates on a list for O(1) swaps; converts back to string for printing.
    T(n) = n * T(n-1) + O(1) => O(n!).
    """
    chars: list[str] = list(s)
    _permutations_helper(chars, start)


def _permutations_helper(chars: list[str], start: int) -> None:
    if start == len(chars):          # base case: all positions filled
        print("".join(chars))
        return
    for i in range(start, len(chars)):
        chars[start], chars[i] = chars[i], chars[start]   # swap
        _permutations_helper(chars, start + 1)             # recurse
        chars[start], chars[i] = chars[i], chars[start]   # swap back (backtrack)
```

**Complexity:** O(n!) time — n choices for position 0, n-1 for position 1, etc. Space O(n) for the recursion stack. Backtracking (swap → recurse → swap back) keeps the array in its original state after each branch, which is the standard technique for in-place permutation generation.

### Exercise 8 — Josephus Problem

n people stand in a circle (indexed 0 to n-1). Starting from person 0, every k-th person is eliminated. Find the index of the last person standing.

> [!TIP]
> After the first elimination, you have a circle of n-1 people. The recursive formula maps positions in the n-1 problem back to positions in the n problem: `jos(n, k) = (jos(n-1, k) + k) % n`. The answer for n=1 is always position 0 (only one person left).

**Solution:**

```python
def josephus(n: int, k: int) -> int:
    """
    Return the 0-indexed position of the last survivor.
    T(n) = T(n-1) + O(1) => O(n) time, O(n) space.
    For large n, convert to iterative to avoid RecursionError.
    """
    if n == 1:
        return 0
    return (josephus(n - 1, k) + k) % n


def josephus_iterative(n: int, k: int) -> int:
    """Same formula, O(n) time, O(1) space. Preferred for large n."""
    pos: int = 0
    for people in range(2, n + 1):
        pos = (pos + k) % people
    return pos
```

**Complexity:** O(n) time. The iterative form uses O(1) space and handles arbitrarily large n without risking `RecursionError`.

## Sources

- Cormen, T.H., Leiserson, C.E., Rivest, R.L., Stein, C. *Introduction to Algorithms* (CLRS), 4th ed. MIT Press, 2022. Chapters 2–4 (algorithms analysis, recurrences, Master Theorem).
- Skiena, S.S. *The Algorithm Design Manual*, 3rd ed. Springer, 2020. Chapter 2 (algorithm analysis).
- Knuth, D.E. *The Art of Computer Programming*, Vol. 1. Chapter 1 (basic concepts and recursion).
- Python documentation: `sys.setrecursionlimit` — https://docs.python.org/3/library/sys.html#sys.setrecursionlimit
- Python documentation: `functools.lru_cache` — https://docs.python.org/3/library/functools.html#functools.lru_cache
- Guido van Rossum on why CPython has no TCO: http://neopythonic.blogspot.com/2009/04/tail-recursion-elimination.html
- Lecture notes: "Analysis of Algorithm", 31 July 2020 (source PDF for this note).

## Related

- [[02-arrays-and-searching]]
- [[12-dynamic-programming]]
