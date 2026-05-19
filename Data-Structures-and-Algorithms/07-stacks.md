---
title: "Stacks"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# Stacks

[toc]

> **TL;DR:** A stack is a linear, LIFO (Last-In First-Out) abstract data type that exposes exactly four operations — push, pop, peek, and isEmpty — at one end called the top. It exists because a large class of problems (expression parsing, recursion elimination, monotone-subsequence queries, bracket matching) reduce cleanly to tracking "what did I see most recently that I haven't yet resolved?", and a stack encodes that question in O(1) per operation. Python's built-in `list` is the idiomatic backing store; for thread-safety or bounded-memory guarantees, `collections.deque` or a linked-list node chain is preferred.

## Vocabulary

- **Stack** — a linear ADT where insertions and deletions happen at a single end called the *top*.
- **LIFO** — Last-In First-Out; the element pushed most recently is the first one popped.
- **Top** — the index or pointer to the most recently inserted element. In an array-backed stack with `n` elements, `top = n - 1`.
- **Push** — insert an element onto the top. O(1) amortized for dynamic arrays, O(1) worst-case for linked lists.
- **Pop** — remove and return the top element. Raises underflow if the stack is empty.
- **Peek (Top)** — return the top element without removing it. Synonym: `top()`.
- **isEmpty** — predicate returning True when the stack has zero elements.
- **Size** — the number of elements currently on the stack.
- **Overflow** — push onto a stack that has reached its fixed capacity (only relevant for fixed-size implementations).
- **Underflow** — pop or peek on an empty stack; must be guarded explicitly.
- **Stack frame** — a block of memory on the program call stack that stores a function's local variables, saved registers, and return address for one invocation.
- **Return address** — the instruction pointer value saved in a stack frame so that when a function returns, execution resumes at the correct call site.
- **Call stack** — the runtime stack maintained by the OS/CPU that holds stack frames for all active function calls; grows downward on x86/ARM by convention.
- **Monotonic stack** — a stack whose elements are maintained in strictly increasing or strictly decreasing order; used to answer "next greater/smaller element" queries in O(n) total.
- **Shunting-yard algorithm** — Dijkstra's algorithm for converting infix expressions to postfix (RPN) using an operator stack, in O(n) time and space.
- **Postfix (RPN)** — Reverse Polish Notation; operators follow their operands so no parentheses or precedence rules are needed during evaluation.
- **Span** — for element i in an array, the number of consecutive elements to its left (including itself) that are less than or equal to arr[i]. Computed efficiently with a monotonic stack.
- **Dynamic stack** — a stack that doubles its backing array when full, giving O(1) amortized push.

## Intuition

Think of a stack of cafeteria trays: you can only add or remove from the top, and the tray you placed last is the one you pick up first. There is no random access — the order of removal is entirely determined by the order of insertion, reversed. This constraint is exactly what makes stacks powerful: whenever a problem has a "most recent unresolved context" structure — an open parenthesis waiting to be closed, a function call waiting to return, a bar in a histogram whose rectangle hasn't been closed yet — a stack tracks it with zero wasted work.

The program call stack is just this idea in hardware. Every function call pushes a frame; every return pops one. Deep recursion is literally a deep stack. When you convert a recursive algorithm to an iterative one using an explicit stack, you are doing manually what the CPU does automatically — and you gain control over stack depth, memory layout, and early termination.

## Memory layout

**Figure: array-backed stack growing upward (top pointer tracks the frontier).**

```
Index:  0     1     2     3     4   ...
      ┌─────┬─────┬─────┬─────┬─────┐
arr:  │ 10  │ 20  │ 30  │     │     │
      └─────┴─────┴─────┴─────┴─────┘
                          ↑
                       top = 2   (0-indexed; 3 elements live)

After push(40):
      ┌─────┬─────┬─────┬─────┬─────┐
arr:  │ 10  │ 20  │ 30  │ 40  │     │
      └─────┴─────┴─────┴─────┴─────┘
                                ↑
                             top = 3

After pop() → 40:
      ┌─────┬─────┬─────┬─────┬─────┐
arr:  │ 10  │ 20  │ 30  │(40) │     │   ← slot logically invalid, top moved left
      └─────┴─────┴─────┴─────┴─────┘
                          ↑
                       top = 2
```

**Figure: program call stack (grows downward in memory on x86/ARM).**

```
High address ───────────────────────────────
             │  main() frame                │
             │    local vars of main        │
             │    return address (OS)       │
             ├──────────────────────────────┤
             │  factorial(n) frame          │
             │    n = 5                     │
             │    return address → main     │
             ├──────────────────────────────┤
             │  factorial(n-1) frame        │
             │    n = 4                     │
             │    return address → fact(5)  │
             ├──────────────────────────────┤
             │  factorial(n-2) frame        │  ← SP (stack pointer) lives here
             │    n = 3                     │
             │    return address → fact(4)  │
Low address  ───────────────────────────────
             ↓  stack grows downward
```

Each frame is allocated by decrementing the stack pointer (SP) on entry and restored by incrementing SP on return. This is why unbounded recursion causes a stack overflow: SP eventually hits a guard page the OS placed at the bottom of the stack segment.
## Math foundations

Stacks look simple enough that the interesting math is easy to skip — but several non-obvious results live here. This section covers five: the three classical proofs of O(1) amortized push, the formal potential-method argument, the combinatorial count of valid sequences via Catalan numbers, the aggregate bound on monotonic stacks, and the connection between recursion depth and tree structure.

### Amortized analysis — three methods for the dynamic-array stack

Amortized analysis answers "what is the *average* cost per operation over a sequence of n operations?" even when individual operations have wildly different costs. There are three canonical methods, and the dynamic-array stack (which doubles capacity on overflow) is the standard worked example for all three.

**Aggregate method.** Count the total work T(n) for n pushes from scratch, then divide. Every push pays 1 unit of work. Each resize at capacity $2^k$ copies $2^k$ elements. The total copy work across all resizes for n pushes is bounded by the geometric series:

```math
1 + 2 + 4 + \cdots + 2^k \leq 2 \cdot 2^k \leq 2n
```

So $T(n) \leq n + 2n = 3n$, giving amortized cost $T(n)/n \leq 3 = O(1)$.

**Accounting method (banker's method).** Assign each push a *charge* of 3 credits: 1 to pay for itself, 1 to pre-pay for copying itself at the next resize, and 1 to pre-pay for copying some "old" element. The invariant is that each element currently in the stack carries at least 2 unused credits. When a resize occurs, every element being copied can pay for itself from its stored credits, so the resize costs 0 against the credit bank. Since no operation ever goes into debt, 3 credits per push is sufficient — O(1) amortized.

**Potential method.** Define a potential function $\Phi$ that maps the data structure state to a non-negative real. The amortized cost of operation $i$ is:

```math
\hat{c}_i = c_i + \Phi(S_i) - \Phi(S_{i-1})
```

where $c_i$ is the actual cost and $\Delta\Phi = \Phi(S_i) - \Phi(S_{i-1})$ is the change in potential. Summing over $n$ operations gives $\sum \hat{c}_i = \sum c_i + \Phi(S_n) - \Phi(S_0)$. If $\Phi(S_n) \geq \Phi(S_0)$, then $\sum c_i \leq \sum \hat{c}_i$ — the amortized total is an upper bound on actual total cost.

> [!IMPORTANT]
> The potential method only proves a valid amortized bound when $\Phi(S_0) = 0$ and $\Phi(S_i) \geq 0$ for all $i$. If either condition is violated, the telescoping sum does not give a valid upper bound on actual cost.

### The potential method, formally

For the doubling stack, define the potential as twice the number of elements minus the current capacity:

```math
\Phi(S) = 2 \cdot \text{size}(S) - \text{capacity}(S)
```

Verify the boundary conditions. At the start: $\Phi(S_0) = 2 \cdot 0 - 1 = -1$... actually, for the analysis to work cleanly, set initial capacity to 1 and initial size to 0, giving $\Phi = 2(0) - 1 = -1$. This is a standard textbook friction point — CLRS uses $\Phi = 2 \cdot \text{size} - \text{capacity}$ with capacity initialized to 1 and proves $\Phi \geq 0$ holds after the first push. The key case is a **push that triggers a resize**:

Suppose immediately before the resize, size = capacity = $m$. Then $\Phi_{\text{before}} = 2m - m = m$. After the resize: new capacity = $2m$, new size = $m + 1$, so $\Phi_{\text{after}} = 2(m+1) - 2m = 2$. The actual cost is $c_i = m + 1$ (copy $m$ elements plus write the new one). The amortized cost is:

```math
\hat{c}_i = c_i + \Delta\Phi = (m+1) + (2 - m) = 3
```

The potential drop of $m - 2$ exactly pays for the $m$ copies. The amortized cost is 3, regardless of $m$ — O(1). For a **push that does not trigger a resize**, $c_i = 1$, size increases by 1, capacity unchanged, so $\Delta\Phi = 2$, giving $\hat{c}_i = 1 + 2 = 3$. Again O(1).

> [!NOTE]
> Different growth factors change the constant, not the asymptotic. A growth factor of $r > 1$ gives $\Phi = \frac{r}{r-1} \cdot \text{size} - \text{capacity}$ (scaled to make the resize amortized cost a constant). Python's `list` uses $r \approx 1.125$ for large lists, which reduces memory waste at the cost of more frequent resizes — still O(1) amortized.

### Catalan numbers and stack-sortable sequences

A sequence of $n$ distinct items arrives one at a time; you may push each item onto a stack or output it directly; you may pop from the stack at any time. The number of distinct output orderings (valid stack sequences) equals the $n$-th Catalan number:

```math
C_n = \frac{1}{n+1}\binom{2n}{n} = \frac{(2n)!}{(n+1)!\, n!}
```

The first few values: $C_0=1$, $C_1=1$, $C_2=2$, $C_3=5$, $C_4=14$, $C_5=42$. This same number appears in at least a dozen other combinatorial contexts that all reduce to the same recurrence $C_n = \sum_{k=0}^{n-1} C_k C_{n-1-k}$ with $C_0 = 1$:

| Structure counted | $n$ refers to |
| :--- | :--- |
| Valid parenthesizations of $n+1$ factors | $n$ pairs of parentheses |
| Full binary trees with $n+1$ leaves | $n$ internal nodes |
| Valid stack-sortable permutations of $\{1,\ldots,n\}$ | $n$ items |
| Triangulations of a convex $(n+2)$-gon | $n$ diagonals |
| Monotonic lattice paths from $(0,0)$ to $(n,n)$ not crossing the diagonal | $n$ steps each direction |

The combinatorial fact that $C_n$ grows as $4^n / (n^{3/2} \sqrt{\pi})$ means the number of valid stack sequences grows exponentially — nearly all permutations are achievable via a stack for large $n$, but exactly $n! - C_n$ are not (those require at least one "321" pattern, which forces a pop-before-push violation).

> [!NOTE]
> A permutation is **stack-sortable** (passable through one stack to produce sorted output) if and only if it avoids the pattern 231. This characterization, due to Knuth (TAOCP Vol. 1, Section 2.2.1), directly connects the Catalan number count to forbidden permutation patterns.

### Monotonic stack — amortized linear bound

A monotonic stack processes $n$ elements and may pop many elements per step (the inner while-loop). Naive reasoning says worst-case O(n) pops per step → O(n²) total. The aggregate method proves the actual bound is O(n).

The argument is the same "charge each element once" observation: every element is **pushed exactly once** and **popped at most once**. The total number of push operations is $n$ (one per element). The total number of pop operations is at most $n$ (since you cannot pop an element that was never pushed). Therefore:

```math
\text{total work} = \text{total pushes} + \text{total pops} \leq n + n = 2n = O(n)
```

More formally: define a token accounting where each element pays 2 tokens on push — 1 for the push and 1 pre-paid for its eventual pop. Since every pop is pre-paid, the stack never goes into debt, and the total token expenditure is $2n$. This is the aggregate method applied to a two-cost operation.

> [!TIP]
> When analyzing any algorithm with a "pop until condition" inner loop — sliding window, monotonic stack, union-find amortization — check whether each element can be "charged" on insertion rather than on deletion. If yes, the total cost collapses from O(n × worst-case-inner) to O(n).

### Stack depth and recursion

The maximum stack depth during a computation equals the maximum recursion depth of the corresponding call tree. For a recursion tree with $n$ nodes, depth depends on shape:

```math
\text{depth} = \begin{cases}
O(\log n) & \text{if the tree is balanced (each internal node has } \geq 2 \text{ children, heights differ by } \leq 1\text{)} \\
O(n) & \text{if the tree is skewed (each internal node has exactly 1 child)}
\end{cases}
```

This is not a coincidence — it mirrors the structure of mathematical induction. A proof by strong induction on $n$ with a binary split at $n/2$ corresponds to a balanced recursion tree with $O(\log n)$ depth. A proof that recurses on $n-1$ (e.g., factorial, naive list reversal) corresponds to a skewed tree with $O(n)$ depth.

The practical implication: CPython's default recursion limit is 1000 frames. Mergesort on 10,000 elements needs $O(\log 10000) \approx 13$ frames — safe. Naive factorial(5000) needs 5000 frames — crashes. The inequality to check before writing a recursive solution:

```math
\text{max depth} \leq \lfloor \log_2 n \rfloor + 1 \implies \text{safe in CPython at default limit for } n \leq 2^{999} \approx 10^{300}
```

For skewed trees: $\text{max depth} = n$, so the recursion limit is hit at $n = 1000$, which is essentially every practical input.

> [!WARNING]
> CPython's recursion limit is a frame count, not a byte count. A single frame that allocates a large NumPy array does not count extra against the limit — but it can exhaust the OS thread stack (typically 8 MB on Linux, 512 KB on macOS for non-main threads) independently of the Python frame counter. Both limits apply simultaneously.


## How it works

The stack ADT has two canonical backing structures: a dynamic array and a singly-linked list. Both provide O(1) amortized (array) or O(1) worst-case (linked list) push and pop. The choice matters in practice: arrays give better cache locality; linked lists give true O(1) worst-case at the cost of a heap allocation per push.

### Array-backed stack

An array-backed stack stores elements in a contiguous block and keeps a `top` index pointing to the last valid element (initialized to -1 for empty). Push increments `top` and writes; pop reads and decrements `top`. When the backing array is full, a dynamic implementation allocates a new array of double the capacity, copies all elements, and continues — the amortized cost of this doubling strategy is O(1) per push. Fixed-size implementations (embedded systems, interview questions) skip the resize and raise overflow instead.

```python
from typing import Generic, TypeVar, Optional

T = TypeVar("T")

class ArrayStack(Generic[T]):
    """LIFO stack backed by a Python list (dynamic array). O(1) amortized push/pop."""

    def __init__(self) -> None:
        self._data: list[T] = []

    def push(self, item: T) -> None:
        self._data.append(item)          # list.append is O(1) amortized

    def pop(self) -> T:
        if self.is_empty():
            raise IndexError("pop from empty stack")
        return self._data.pop()          # list.pop() is O(1)

    def peek(self) -> T:
        if self.is_empty():
            raise IndexError("peek at empty stack")
        return self._data[-1]

    def is_empty(self) -> bool:
        return len(self._data) == 0

    def size(self) -> int:
        return len(self._data)

    def __repr__(self) -> str:
        return f"ArrayStack({self._data})"
```

> [!TIP]
> In Python, `list.append` and `list.pop()` (no index) operate on the tail of the list, which maps perfectly to the top of a stack. Never use `list.pop(0)` for stack operations — that is O(n). Also never use `list.insert(0, x)` for push — same O(n) cost. The tail IS the top.

### Linked-list-backed stack

A linked-list stack stores each element in a heap-allocated node that holds a value and a `next` pointer. The `head` of the list is the top of the stack. Push allocates a new node and prepends it; pop removes the head. No capacity limit, no amortized awkwardness, but each push triggers a heap allocation which can hurt cache performance at high throughput.

```python
from __future__ import annotations
from typing import Generic, TypeVar, Optional

T = TypeVar("T")

class _Node(Generic[T]):
    __slots__ = ("val", "next")

    def __init__(self, val: T, nxt: Optional[_Node[T]] = None) -> None:
        self.val = val
        self.next = nxt


class LinkedStack(Generic[T]):
    """LIFO stack backed by a singly-linked list. True O(1) push/pop."""

    def __init__(self) -> None:
        self._head: Optional[_Node[T]] = None
        self._size: int = 0

    def push(self, item: T) -> None:
        self._head = _Node(item, self._head)
        self._size += 1

    def pop(self) -> T:
        if self._head is None:
            raise IndexError("pop from empty stack")
        val = self._head.val
        self._head = self._head.next
        self._size -= 1
        return val

    def peek(self) -> T:
        if self._head is None:
            raise IndexError("peek at empty stack")
        return self._head.val

    def is_empty(self) -> bool:
        return self._head is None

    def size(self) -> int:
        return self._size
```

### Monotonic stack

The monotonic stack pattern maintains a stack whose elements satisfy a monotone order invariant (all increasing or all decreasing from bottom to top). When a new element arrives, we pop all elements that violate the invariant before pushing the new one. The key insight is that each element is pushed and popped at most once across the entire sweep, giving O(n) total work even though there is an inner while-loop.

The canonical application is **next greater element**: for every index i, find the nearest j > i such that arr[j] > arr[i]. The stack holds indices of elements whose "next greater" has not yet been found. When arr[j] arrives and is larger than arr[stack.top()], the stack element at top() has found its answer.

```
Figure: next-greater-element scan over [2, 1, 5, 6, 2, 3].

i=0: push 0           stack: [0]         (element 2, awaiting)
i=1: 1 < 2, push 1    stack: [0, 1]      (1 also awaiting)
i=2: 5 > arr[1]=1 → pop 1, NGE[1]=5
     5 > arr[0]=2 → pop 0, NGE[0]=5
     push 2            stack: [2]
i=3: 6 > arr[2]=5 → pop 2, NGE[2]=6
     push 3            stack: [3]
i=4: 2 < 6, push 4    stack: [3, 4]
i=5: 3 > arr[4]=2 → pop 4, NGE[4]=3
     3 < 6, push 5    stack: [3, 5]
end: remaining [3, 5] → NGE = -1 (none found)
Result: NGE = [5, 5, 6, -1, 3, -1]
```

```python
def next_greater_element(nums: list[int]) -> list[int]:
    """Return NGE[i] = next element greater than nums[i], or -1 if none."""
    n = len(nums)
    result = [-1] * n
    stack: list[int] = []  # stores indices

    for i in range(n):
        # pop all indices whose element is smaller than nums[i]
        while stack and nums[stack[-1]] < nums[i]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```

### Infix to postfix (Shunting-yard)

The Shunting-yard algorithm converts an infix expression like `3 + 4 * 2 / (1 - 5)` to postfix `3 4 2 * 1 5 - / +` in a single left-to-right pass. It uses one operator stack and produces the output stream in-order. The rules encode operator precedence and associativity without any recursive grammar parsing.

**Operator precedence table:**

| Operator | Precedence | Associativity |
| :---: | :---: | :---: |
| `(` `)` | — | — |
| `+` `-` | 1 | Left |
| `*` `/` | 2 | Left |
| `^` (power) | 3 | Right |

**Algorithm rules, applied per token:**

1. **Operand (number/variable):** append to output.
2. **Left parenthesis `(`:** push onto operator stack.
3. **Right parenthesis `)`:** pop and output operators until `(` is found; discard the `(`.
4. **Operator `op`:** while the stack is non-empty, the top is not `(`, and the top has higher precedence than `op` (or equal precedence and `op` is left-associative), pop and output. Then push `op`.
5. **End of input:** pop and output remaining operators.

**Step-by-step trace for `3 + 4 * 2 / (1 - 5)`:**

```
Token  Action                     Stack        Output
─────  ─────────────────────────  ───────────  ─────────────
3      operand → output           []           3
+      push                       [+]          3
4      operand → output           [+]          3 4
*      prec(*) > prec(+), push    [+ *]        3 4
2      operand → output           [+ *]        3 4 2
/      prec(/) = prec(*), pop *   [+]          3 4 2 *
       prec(/) > prec(+), push /  [+ /]        3 4 2 *
(      push                       [+ / (]      3 4 2 *
1      operand → output           [+ / (]      3 4 2 * 1
-      top is (, push             [+ / ( -]    3 4 2 * 1
5      operand → output           [+ / ( -]    3 4 2 * 1 5
)      pop until (: pop -, discard ( [+ /]     3 4 2 * 1 5 -
END    pop /                      [+]          3 4 2 * 1 5 - /
       pop +                      []           3 4 2 * 1 5 - / +
```

Result: `3 4 2 * 1 5 - / +`

```python
def infix_to_postfix(expression: str) -> str:
    """Convert infix expression string to postfix (RPN) using Shunting-yard."""
    precedence: dict[str, int] = {"+": 1, "-": 1, "*": 2, "/": 2, "^": 3}
    right_assoc: set[str] = {"^"}
    output: list[str] = []
    op_stack: list[str] = []

    tokens = expression.split()
    for token in tokens:
        if token.lstrip("-").isdigit() or (token.replace(".", "", 1).isdigit()):
            output.append(token)
        elif token == "(":
            op_stack.append(token)
        elif token == ")":
            while op_stack and op_stack[-1] != "(":
                output.append(op_stack.pop())
            if not op_stack:
                raise ValueError("Mismatched parentheses")
            op_stack.pop()  # discard "("
        elif token in precedence:
            while (
                op_stack
                and op_stack[-1] != "("
                and op_stack[-1] in precedence
                and (
                    precedence[op_stack[-1]] > precedence[token]
                    or (
                        precedence[op_stack[-1]] == precedence[token]
                        and token not in right_assoc
                    )
                )
            ):
                output.append(op_stack.pop())
            op_stack.append(token)
        else:
            raise ValueError(f"Unknown token: {token!r}")

    while op_stack:
        top = op_stack.pop()
        if top == "(":
            raise ValueError("Mismatched parentheses")
        output.append(top)

    return " ".join(output)


def eval_postfix(expression: str) -> float:
    """Evaluate a postfix (RPN) expression string."""
    stack: list[float] = []
    ops: dict[str, object] = {
        "+": float.__add__,
        "-": float.__sub__,
        "*": float.__mul__,
        "/": float.__truediv__,
    }
    for token in expression.split():
        if token in ops:
            if len(stack) < 2:
                raise ValueError("Malformed postfix expression")
            b = stack.pop()
            a = stack.pop()
            stack.append(ops[token](a, b))  # type: ignore[operator]
        else:
            stack.append(float(token))
    if len(stack) != 1:
        raise ValueError("Malformed postfix expression")
    return stack[0]
```

> [!IMPORTANT]
> The Shunting-yard algorithm only works correctly when equal-precedence, left-associative operators cause a pop before the new operator is pushed. If you forget the `token not in right_assoc` check for `^`, you will silently get wrong answers for `2 ^ 3 ^ 2` (should be 512 for right-to-left, not 64).

## Math

### Amortized O(1) push for dynamic arrays

A dynamic stack doubles its capacity each time it is full. Let T(n) be the total cost of n pushes starting from an empty stack of initial capacity 1.

```math
T(n) = n + \sum_{k=0}^{\lfloor \log_2 n \rfloor} 2^k \leq n + 2n = 3n
```

The summation accounts for all resize-and-copy operations: after 1, 2, 4, 8, … elements the array copies 1, 2, 4, 8, … elements respectively. The geometric series sums to at most 2n. Therefore each push costs at most 3 operations amortized — O(1) amortized.

```math
\text{amortized cost per push} = \frac{T(n)}{n} \leq 3 = O(1)
```

### Operation complexity summary

| Operation | Array-backed (dynamic) | Array-backed (fixed) | Linked-list |
| :--- | :---: | :---: | :---: |
| push | O(1) amortized | O(1) | O(1) |
| pop | O(1) | O(1) | O(1) |
| peek | O(1) | O(1) | O(1) |
| isEmpty / size | O(1) | O(1) | O(1) |
| Space | O(n) | O(capacity) | O(n) |

> [!NOTE]
> Python's `list` uses a growth factor slightly above 1 (approximately 1.125 for large lists, not exactly 2), so the amortized constant is tighter in practice — but the asymptotic O(1) amortized argument still holds.

## Real-world example

### Browser back button and undo/redo

The back-button history in a browser is a stack: each page visited is pushed; pressing Back pops it. Undo/redo in an editor uses two stacks: undo-stack and redo-stack. An edit pushes onto undo; pressing Ctrl+Z pops from undo and pushes onto redo; pressing Ctrl+Y pops from redo and pushes back onto undo.

```python
from collections import deque
from typing import Optional

class UndoRedoBuffer:
    """Classic two-stack undo/redo using deque for O(1) amortized appends."""

    def __init__(self) -> None:
        self._undo: deque[str] = deque()
        self._redo: deque[str] = deque()

    def do(self, action: str) -> None:
        """Record a new action; clears the redo history."""
        self._undo.append(action)
        self._redo.clear()

    def undo(self) -> Optional[str]:
        """Undo the most recent action and return it."""
        if not self._undo:
            return None
        action = self._undo.pop()
        self._redo.append(action)
        return action

    def redo(self) -> Optional[str]:
        """Redo the most recently undone action and return it."""
        if not self._redo:
            return None
        action = self._redo.pop()
        self._undo.append(action)
        return action


# Demo
buf = UndoRedoBuffer()
buf.do("type 'H'")
buf.do("type 'e'")
buf.do("type 'l'")
print(buf.undo())   # "type 'l'"
print(buf.undo())   # "type 'e'"
print(buf.redo())   # "type 'e'"
```

> [!TIP]
> Use `collections.deque` instead of `list` when you need thread-safe append/pop from either end. `deque.append` and `deque.pop` are GIL-protected atomic operations in CPython; `list.append` / `list.pop()` are also safe in CPython's GIL model but `deque` is the explicit choice when thread-safety of the data structure itself (not the application logic) matters.

### Function call stack (recursion → explicit stack)

Deep recursion on Python's default 1000-frame limit will crash. Converting to an explicit stack gives you control over depth and memory.

```python
import sys

def factorial_recursive(n: int) -> int:
    """Crashes for n > ~990 on default CPython settings."""
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)

def factorial_iterative(n: int) -> int:
    """Explicit stack mimics the call stack. No recursion limit."""
    call_stack: list[int] = []
    result = 1
    for i in range(2, n + 1):
        call_stack.append(i)
    while call_stack:
        result *= call_stack.pop()
    return result

# Balanced parentheses (classic stack application)
def is_balanced(s: str) -> bool:
    """Return True if every bracket in s has a matching closing bracket."""
    matching: dict[str, str] = {")": "(", "]": "[", "}": "{"}
    stack: list[str] = []
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in ")]}":
            if not stack or stack[-1] != matching[ch]:
                return False
            stack.pop()
    return len(stack) == 0

assert is_balanced("({[]})")
assert not is_balanced("({[}])")
```

## In practice

**Python `list` is the standard stack.** `list.append(x)` and `list.pop()` are both O(1) amortized and written in C. There is no `Stack` class to import in the standard library — you just use `list` directly, or wrap it if you want the ADT interface enforced (e.g. to prevent accidental `list[i]` random access).

**`collections.deque` for thread-safety.** If multiple threads share a stack, `deque` is safer. `deque.appendleft` / `deque.popleft` also let you treat the left end as the top if needed, with the same O(1) amortized cost.

**`sys.setrecursionlimit`.** Python defaults to 1000 frames. Deep tree traversals and graph DFS on large inputs will hit this. Common responses: (a) call `sys.setrecursionlimit(10_000)` at startup (raises the soft limit, not safe for truly deep recursion), or (b) convert to an explicit stack loop (the correct production fix).

**Stack to heap conversion.** Any recursive algorithm can be converted to an iterative one with an explicit stack. The pattern is: maintain a stack of (node, state) tuples where state encodes "how far along this call I am." This is exactly what the CPU does with stack frames. In Python this is especially important for DFS on graphs with millions of nodes.

> [!WARNING]
> `sys.setrecursionlimit(10_000)` does NOT actually give you 10,000 safe frames. CPython uses about 1KB per frame; 10,000 frames = 10MB of C stack, which is close to the OS thread stack limit (typically 8MB on Linux). You can trigger a segfault — not a clean Python `RecursionError`. For deep recursion, convert to an explicit stack.

**The "two stacks in one array" trick.** When you need two stacks that share a fixed memory buffer (embedded, contest), initialize stack 1 from index 0 growing right, and stack 2 from index n-1 growing left. Overflow is detected when `top1 + 1 == top2`. This achieves optimal space utilization — neither stack wastes its "half."

**Monotonic stack patterns to memorize:**

| Problem pattern | Stack invariant | Complexity |
| :--- | :--- | :---: |
| Next greater element | Decreasing (bottom→top) | O(n) |
| Next smaller element | Increasing (bottom→top) | O(n) |
| Largest rectangle in histogram | Increasing heights | O(n) |
| Daily temperatures | Decreasing temperatures | O(n) |
| Stock span | Decreasing prices | O(n) |
| Trapping rain water | Decreasing heights | O(n) |

> [!NOTE]
> The "stock span" problem (how many consecutive days before today had price ≤ today's price) is exactly a monotonic stack application: maintain a stack of (price, span) pairs where prices are decreasing. When today's price exceeds the top, pop and accumulate the span. The span for each day is computed in O(1) amortized.

## Pitfalls

- **Using a list but calling `pop(0)` or `insert(0, x)`.** This is O(n) because Python must shift all elements. The correct stack operations are `append` and `pop()` (no index). If you find yourself using `pop(0)`, you want a queue — see `collections.deque` with `popleft`.
- **Forgetting the underflow check.** `list.pop()` raises `IndexError` on an empty list, but `list[-1]` (for peek) returns the last element of an empty list — no, wait: it raises `IndexError` too. Both are fine in Python but should be caught and re-raised as domain-specific exceptions rather than leaking `IndexError` to callers.
- **Assuming stack = FIFO.** LIFO and FIFO are opposites. A stack is not a queue. If you push 1, 2, 3 and then pop three times you get 3, 2, 1 — not 1, 2, 3. This seems obvious in isolation but gets confused under time pressure in interviews.
- **Thinking `sys.setrecursionlimit` is safe at large values.** See the "In practice" note above. A segfault is possible; the fix is an explicit stack, not a higher limit.
- **Mutating the backing list while iterating over it.** If you do `for x in my_list_stack` and push or pop inside the loop, you corrupt the iteration. Use indices or copy the list first.
- **Balanced parentheses: checking only count, not order.** `)(` has count 2 open and 2 close but is not balanced. You must use a stack, not just a counter.

> [!CAUTION]
> In the Min Stack problem (O(1) getMin), the naive approach stores the minimum as a single class variable. This is WRONG when popping the current minimum — the minimum may change and you have no record of what the previous minimum was. Always use a parallel auxiliary stack that stores the minimum valid at each push level.

## Exercises

### Exercise 1 — Valid Parentheses (LeetCode 20)

Given a string containing only `(`, `)`, `{`, `}`, `[`, `]`, return True if the brackets are balanced. Every open bracket must close in correct order.

**Hint:**

> [!TIP]
> Use a stack. Push every open bracket. On every close bracket, check that the stack top is the matching open bracket. If the stack is empty at that point, or the bracket doesn't match, return False. At the end, the stack must be empty.

**Time:** O(n). **Space:** O(n).

```python
def is_valid(s: str) -> bool:
    matching: dict[str, str] = {")": "(", "]": "[", "}": "{"}
    stack: list[str] = []
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in ")]}":
            if not stack or stack[-1] != matching[ch]:
                return False
            stack.pop()
    return not stack
```

---

### Exercise 2 — Min Stack (LeetCode 155)

Design a stack that supports push, pop, top, and getMin — all in O(1) time.

**Hint:**

> [!TIP]
> Maintain a parallel auxiliary stack `min_stack`. On every push(x), push `min(x, min_stack.top())` onto `min_stack`. On every pop, pop from both stacks simultaneously. getMin returns `min_stack.top()`.

**Time:** O(1) all operations. **Space:** O(n).

```python
class MinStack:
    def __init__(self) -> None:
        self._data: list[int] = []
        self._min: list[int] = []  # parallel min-tracking stack

    def push(self, val: int) -> None:
        self._data.append(val)
        current_min = val if not self._min else min(val, self._min[-1])
        self._min.append(current_min)

    def pop(self) -> None:
        self._data.pop()
        self._min.pop()

    def top(self) -> int:
        return self._data[-1]

    def get_min(self) -> int:
        return self._min[-1]
```

---

### Exercise 3 — Next Greater Element I (LeetCode 496)

Given two arrays `nums1` and `nums2` where `nums1` is a subset of `nums2`, find the next greater element for each element of `nums1` in `nums2`.

**Hint:**

> [!TIP]
> Build a NGE map over `nums2` using a monotonic decreasing stack. Then look up each `nums1[i]` in the map.

**Time:** O(n + m). **Space:** O(n).

```python
def next_greater_element(nums1: list[int], nums2: list[int]) -> list[int]:
    nge: dict[int, int] = {}
    stack: list[int] = []

    for num in nums2:
        while stack and stack[-1] < num:
            nge[stack.pop()] = num
        stack.append(num)
    # remaining elements have no greater element
    while stack:
        nge[stack.pop()] = -1

    return [nge[n] for n in nums1]
```

---

### Exercise 4 — Daily Temperatures (LeetCode 739)

Given an array `temperatures`, return an array `answer` where `answer[i]` is the number of days until a warmer temperature. If no future warmer day exists, `answer[i] = 0`.

**Hint:**

> [!TIP]
> Monotonic decreasing stack of indices. When a warmer day arrives at index j, every index in the stack with a cooler temperature gets its answer: `j - stack_index`.

**Time:** O(n). **Space:** O(n).

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    answer = [0] * n
    stack: list[int] = []  # indices with unresolved warmer day

    for i, temp in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < temp:
            prev = stack.pop()
            answer[prev] = i - prev
        stack.append(i)

    return answer
```

---

### Exercise 5 — Largest Rectangle in Histogram (LeetCode 84)

Given an array `heights` representing bar heights in a histogram, find the area of the largest rectangle.

**Hint:**

> [!TIP]
> Monotonic increasing stack of indices. When a shorter bar arrives at i (or we reach the end), every bar popped has the arriving bar as its right boundary and the new stack top as its left boundary. Width = right_boundary - left_boundary - 1.

**Time:** O(n). **Space:** O(n).

```python
def largest_rectangle_area(heights: list[int]) -> int:
    stack: list[int] = []  # monotonic increasing by height
    max_area = 0
    heights = heights + [0]  # sentinel to flush the stack at the end

    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] >= h:
            height = heights[stack.pop()]
            width = i if not stack else i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    return max_area
```

---

### Exercise 6 — Evaluate Reverse Polish Notation (LeetCode 150)

Evaluate an expression given in postfix notation. Operands are integers; operators are `+`, `-`, `*`, `/` (integer division truncating toward zero).

**Hint:**

> [!TIP]
> Push numbers onto a stack. On each operator, pop two operands (second pop is left operand), apply the operator, push the result.

**Time:** O(n). **Space:** O(n).

```python
def eval_rpn(tokens: list[str]) -> int:
    stack: list[int] = []
    for token in tokens:
        if token in ("+", "-", "*", "/"):
            b = stack.pop()
            a = stack.pop()
            if token == "+":
                stack.append(a + b)
            elif token == "-":
                stack.append(a - b)
            elif token == "*":
                stack.append(a * b)
            else:
                stack.append(int(a / b))  # truncate toward zero
        else:
            stack.append(int(token))
    return stack[0]
```

---

### Exercise 7 — Decode String (LeetCode 394)

Given an encoded string like `3[a2[c]]`, decode it to `accaccacc`. Numbers can be multi-digit; brackets can be nested.

**Hint:**

> [!TIP]
> Use two stacks: one for repeat counts and one for string prefixes. On `[`, push the current string and current number onto their stacks, reset both. On `]`, pop and repeat.

**Time:** O(n * max_k) where max_k is the maximum repeat count. **Space:** O(n).

```python
def decode_string(s: str) -> str:
    count_stack: list[int] = []
    str_stack: list[str] = []
    current_str = ""
    current_num = 0

    for ch in s:
        if ch.isdigit():
            current_num = current_num * 10 + int(ch)
        elif ch == "[":
            count_stack.append(current_num)
            str_stack.append(current_str)
            current_num = 0
            current_str = ""
        elif ch == "]":
            repeats = count_stack.pop()
            prefix = str_stack.pop()
            current_str = prefix + current_str * repeats
        else:
            current_str += ch

    return current_str
```

---

### Exercise 8 — Implement Queue Using Two Stacks (LeetCode 232)

Implement a FIFO queue using only two stacks, so that push is O(1) amortized and pop/peek are O(1) amortized.

**Hint:**

> [!TIP]
> Stack `s1` accepts all pushes. Stack `s2` is the "reversed" view for pops. When `s2` is empty and a pop is needed, pour all of `s1` into `s2` (reversing the order). Each element is moved at most once across the stacks' lifetimes.

**Time:** O(1) amortized push, O(1) amortized pop. **Space:** O(n).

```python
class MyQueue:
    def __init__(self) -> None:
        self._inbox: list[int] = []   # push here
        self._outbox: list[int] = []  # pop from here

    def push(self, x: int) -> None:
        self._inbox.append(x)

    def _transfer(self) -> None:
        if not self._outbox:
            while self._inbox:
                self._outbox.append(self._inbox.pop())

    def pop(self) -> int:
        self._transfer()
        return self._outbox.pop()

    def peek(self) -> int:
        self._transfer()
        return self._outbox[-1]

    def empty(self) -> bool:
        return not self._inbox and not self._outbox
```

---

### Exercise 9 — Sort a Stack Using Recursion

Sort a stack in ascending order (smallest on top) using only recursion and the stack's own operations — no auxiliary data structure allowed.

**Hint:**

> [!TIP]
> Pop the top, recursively sort the rest, then insert the popped element into the correct sorted position (which itself requires a recursive insert helper).

**Time:** O(n^2). **Space:** O(n) call stack.

```python
def sort_stack(stack: list[int]) -> None:
    """Sort stack in-place (smallest on top) using recursion only."""
    if not stack:
        return
    top = stack.pop()
    sort_stack(stack)
    _insert_sorted(stack, top)


def _insert_sorted(stack: list[int], val: int) -> None:
    """Insert val into a sorted stack at the correct position."""
    if not stack or stack[-1] <= val:
        stack.append(val)
        return
    top = stack.pop()
    _insert_sorted(stack, val)
    stack.append(top)
```

## Sources

- Cormen, T.H. et al. *Introduction to Algorithms*, 4th ed. (CLRS). Chapter 10.1 (Stacks and Queues).
- Dijkstra, E.W. "Algol 60 Translation." — original description of the Shunting-yard algorithm.
- Python documentation: [`list`](https://docs.python.org/3/library/stdtypes.html#list), [`collections.deque`](https://docs.python.org/3/library/collections.html#collections.deque), [`sys.setrecursionlimit`](https://docs.python.org/3/library/sys.html#sys.setrecursionlimit).
- LeetCode problems 20, 84, 150, 155, 232, 394, 496, 739.
- Conversation with user on 2026-05-19.

## Related

- [[01-intro-and-recursion]]
- [[06-linked-list]]
- [[08-queues]]
- [[09-trees]]
- [[11-greedy-and-backtracking]]
