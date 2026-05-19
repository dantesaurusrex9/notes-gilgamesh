---
title: "Arrays and Searching"
created: 2026-05-19
updated: 2026-05-19
tags: [dsa, arrays, searching, binary-search, dynamic-array, kadane, boyer-moore, prefix-sum, sliding-window]
aliases: ["02-arrays-and-searching", "array", "binary search"]
---

# Arrays and Searching

[toc]

> **TL;DR:** An array is a contiguous block of memory holding elements of the same type, giving O(1) random access via arithmetic on a base address. Python's `list` is a dynamic array — a pointer array that doubles its capacity on overflow, giving amortized O(1) append. Searching that array naively costs O(n); if the array is sorted, binary search cuts it to O(log n) by halving the live search interval at every step.

## Vocabulary

**Array** — a fixed-size, contiguous block of memory storing elements of a single type, indexed from 0.

**Dynamic array** — an array that owns a heap-allocated buffer larger than its current length; when the buffer fills, it allocates a new buffer (typically 2× the old capacity), copies all elements, and frees the old buffer.

**Capacity** — the number of slots currently allocated in the backing buffer of a dynamic array.

**Length** — the number of elements currently stored (always ≤ capacity).

**Amortized complexity** — the average cost per operation over a sequence of operations; individual ops may be expensive, but the amortized cost is lower. For dynamic array append: each element is moved at most O(log n) times across all doublings, giving amortized O(1) per append.

**Locality of reference** — the tendency of a program to access nearby memory addresses. Arrays exploit spatial locality: loading one cache line (64 bytes on x86) pulls in several adjacent elements.

**Row-major order** — 2D array layout where a complete row is stored contiguously before the next row starts (C, Python/NumPy default).

**Column-major order** — 2D array layout where a complete column is stored contiguously (Fortran, MATLAB default).

**Sentinel** — a special value placed at the boundary of a data structure to eliminate an explicit bounds check inside a loop; e.g., placing the target value at index n so the linear search always terminates without a separate `i < n` check.

**Invariant** — a predicate that is true at the start and end of every loop iteration; writing one down for binary search (`arr[lo-1] < target <= arr[hi+1]` for lower-bound) is the cleanest way to avoid off-by-one bugs.

**Linear search** — scan every element from left to right until the target is found or the array is exhausted; O(n) worst case.

**Binary search** — repeatedly bisect a sorted array by comparing the midpoint to the target and discarding the half that cannot contain the target; O(log n).

**Lower bound** — the index of the first element >= target in a sorted array (Python: `bisect_left`).

**Upper bound** — the index of the first element > target in a sorted array (Python: `bisect_right`).

**Peak element** — an element strictly greater than its neighbors (boundary elements are peaks if greater than their single neighbor).

**Kadane's algorithm** — O(n) algorithm for maximum subarray sum; at each index, either extend the previous subarray or start a fresh one.

**Boyer-Moore voting algorithm** — O(n) time, O(1) space algorithm for finding the majority element (appears > n/2 times) via a candidate/count pairing.

**Prefix sum** — an auxiliary array `P` where `P[i]` = sum of `arr[0..i-1]`; enables O(1) range-sum queries after O(n) preprocessing.

**Sliding window** — a technique that maintains a contiguous subarray of fixed or variable length using two pointers, avoiding recomputation by incrementally updating the window's aggregate.

## Intuition

Think of an array as a row of labelled mailboxes nailed to a wall — every box is the same width, so reaching box number `i` is one multiplication away from the start address. Because the boxes are physically adjacent, the post office (CPU prefetcher) loads several boxes into the cache when you open any one of them: that is spatial locality, and it is the single biggest performance advantage arrays have over linked structures.

A dynamic array is the same mailbox row, but built on a collapsible wall. When you run out of boxes, you collapse the wall, build a new one twice as long, move all the mail, and continue. The occasional move is expensive, but because moves happen exponentially less often (at sizes 1, 2, 4, 8, …), the average cost per insertion stays constant.

Binary search is a phone-book problem. Given a sorted phone book, you open to the middle: if the name is earlier, throw away the right half; if later, throw away the left half. Each page-turn halves the remaining book. After at most log₂(n) turns, you either find the name or confirm it isn't there.

## Math foundations

The O(1) indexing and O(log n) search costs that define arrays are not handwaving — they come from three concrete mathematical facts: pointer arithmetic reduces element access to a multiply-add; the logarithm's base is irrelevant to Big-O; and the binary search recurrence telescopes to exactly lg n steps. Amortized analysis then shows that dynamic array growth, despite its O(n) spikes, amortizes to O(1) per append across any sequence of operations.

### Address arithmetic — why indexing is O(1)

Every element in a 1D array has the same width in bytes (sizeof(elem)). The CPU therefore reaches element `A[i]` via a single multiply and add on the base address — no traversal, no loop, no branching. This single-instruction address computation is the physical reason random access is O(1), and it is what fundamentally distinguishes arrays from linked structures.

For a 1D array with element size `s` bytes:

```math
\text{addr}(A[i]) = \text{base} + i \times s
```

For a 2D array of shape (rows × cols) stored in **row-major** order (C, Python/NumPy default), a complete row is laid out contiguously before the next row. Reaching element A[i][j] requires skipping `i` full rows (each of `cols × s` bytes) then `j` more elements:

```math
\text{addr}(A[i][j]) = \text{base} + (i \times \text{cols} + j) \times s
```

For **column-major** order (Fortran, MATLAB), a complete column is contiguous. Reaching A[i][j] requires skipping `j` full columns (each of `rows × s` bytes) then `i` more elements:

```math
\text{addr}(A[i][j]) = \text{base} + (j \times \text{rows} + i) \times s
```

> [!IMPORTANT]
> The only difference between row-major and column-major is which index is the "outer stride." Swapping which loop is the inner loop (i vs. j) can flip cache performance from optimal to pathological — every inner step hits a different cache line in the wrong order.

### Logarithms — base doesn't matter in Big-O

Binary search halves the search space at each step, producing a depth of log₂(n) steps. Ternary search thirds it — log₃(n) steps. An algorithm that divides by k at each level runs to log_k(n) steps. All of these are the same in Big-O because logarithms in different bases differ only by a constant factor.

The change-of-base identity makes this precise. For any two bases a and b greater than 1:

```math
\log_a n = \frac{\log_b n}{\log_b a}
```

The term `1 / log_b(a)` is a constant that depends only on the two bases, not on n. Big-O absorbs constants, so `O(log_2 n) = O(log_3 n) = O(log n)` — the base is irrelevant and conventionally dropped.

> [!NOTE]
> This is why binary search, ternary search, and B-tree lookup (which fans out by hundreds at each node) all have O(log n) search complexity — the branching factor shifts the log's constant but does not change the asymptotic class. The practical difference shows up in real cache miss counts, not in the Big-O label.

### Recurrence T(n) = T(n/2) + c — solved by unrolling

Binary search is described exactly by a divide-and-conquer recurrence: to search n elements, do one O(1) comparison and recurse on n/2 elements. The recurrence is:

```math
T(n) = T\!\left(\tfrac{n}{2}\right) + c, \quad T(1) = c
```

Unrolling this recurrence by hand makes the logarithmic depth explicit. Substitute the definition of T repeatedly until the base case is reached:

```math
\begin{aligned}
T(n) &= T\!\left(\tfrac{n}{2}\right) + c \\
     &= T\!\left(\tfrac{n}{4}\right) + 2c \\
     &= T\!\left(\tfrac{n}{8}\right) + 3c \\
     &\;\vdots \\
     &= T(1) + c \cdot \log_2 n \\
     &= c + c \log_2 n \\
     &= O(\log n)
\end{aligned}
```

After exactly log₂(n) halvings the subproblem reaches size 1. Each level adds exactly c work. Total work is c · log₂(n) + c, which is O(log n).

> [!IMPORTANT]
> The unrolling argument only works cleanly when n is a power of 2. For arbitrary n, the analysis uses floors and ceilings: T(n) = T(⌊n/2⌋) + c. The result is the same — O(log n) — but the exact constant differs by at most 1 level. This matters for tight constant-factor analysis but not for asymptotic classification.

### Master theorem — brief reference

The Master theorem gives a closed-form solution for recurrences of the form T(n) = aT(n/b) + f(n), where a ≥ 1 subproblems of size n/b are created and f(n) is the cost of the work done outside the recursive calls. The three cases compare f(n) to the critical function n^(log_b(a)):

```math
T(n) = a \, T\!\left(\tfrac{n}{b}\right) + f(n)
```

| Case | Condition | Solution |
| :--- | :--- | :--- |
| 1 | f(n) = O(n^(log_b(a) − ε)) for some ε > 0 — work dominated by leaves | T(n) = Θ(n^(log_b(a))) |
| 2 | f(n) = Θ(n^(log_b(a))) — work balanced across all levels | T(n) = Θ(n^(log_b(a)) · log n) |
| 3 | f(n) = Ω(n^(log_b(a) + ε)) for some ε > 0, plus a regularity condition — work dominated by root | T(n) = Θ(f(n)) |

Applying to binary search: a = 1 (one recursive call), b = 2 (half the problem), f(n) = c (constant comparison work). The critical function is n^(log₂(1)) = n⁰ = 1. Since f(n) = c = Θ(1) = Θ(n^(log₂(1))), this falls into **Case 2** and gives T(n) = Θ(log n).

> [!NOTE]
> Merge sort has a = 2, b = 2, f(n) = Θ(n) (the merge step). The critical function is n^(log₂(2)) = n¹ = n. Since f(n) = Θ(n) = Θ(n¹), this is also Case 2, giving T(n) = Θ(n log n). This is previewed in [[03-sorting]].

### Amortized analysis of dynamic arrays

Dynamic array `append` occasionally triggers an O(n) reallocation — copying all n elements to a new buffer of size 2n. Despite this spike, the amortized cost per append is O(1). The aggregate method proves this by counting total work across all n appends rather than analyzing each in isolation.

Starting from an empty array and performing n appends, doublings occur at sizes 1, 2, 4, 8, …, up to n (assuming n is a power of 2). The total number of element copies summed across all doublings is a geometric series:

```math
1 + 2 + 4 + \cdots + \frac{n}{2} = n - 1 < n
```

So across n appends, the total copy work is less than n. Adding the n append operations themselves, total work is less than 2n. Dividing by n operations gives amortized cost strictly less than 2 per append — that is, O(1) amortized.

The geometric series identity used above is:

```math
\sum_{k=0}^{m} 2^k = 2^{m+1} - 1
```

Setting m = log₂(n) − 1 gives the sum 1 + 2 + … + n/2 = n − 1, which is the exact total copy count.

> [!TIP]
> The factor-of-2 growth is not an accident — it is the smallest integer growth factor that guarantees O(1) amortized append. A growth factor of 1.5 also achieves O(1) amortized but with a smaller constant (fewer wasted slots, but more frequent reallocations). CPython uses approximately 1.125 for small lists, trading slightly more frequent reallocations for lower memory waste.

## How it works

Arrays and searching form a layered system: memory layout determines what the hardware can do cheaply, the ADT abstraction determines what operations are semantically correct, and the algorithm layer (linear vs. binary search) sits on top.

### Array memory layout

A 1D array of 32-bit integers lives at a base address `x`. Element `i` is at byte offset `i * 4` from `x`, so the address is `x + i * sizeof(int)`. This is plain pointer arithmetic — O(1) and branch-free. The diagram below shows 5 integers occupying 20 contiguous bytes:

```
Base address x
│
▼
┌──────┬──────┬──────┬──────┬──────┐
│  10  │  20  │  30  │  35  │  45  │   values
├──────┼──────┼──────┼──────┼──────┤
│  x   │ x+4  │ x+8  │ x+12 │ x+16 │   byte addresses (int32)
└──────┴──────┴──────┴──────┴──────┘
  [0]    [1]    [2]    [3]    [4]       indices
```

For a 2D array of shape (rows, cols) in **row-major** order (C/Python), element `A[i][j]` is at:

```math
\text{addr}(A[i][j]) = \text{base} + (i \times \text{cols} + j) \times \text{sizeof(elem)}
```

**Figure:** row-major 2D layout for a 3×4 int array.

```
Row 0: A[0][0] A[0][1] A[0][2] A[0][3]
Row 1: A[1][0] A[1][1] A[1][2] A[1][3]   ← contiguous in memory
Row 2: A[2][0] A[2][1] A[2][2] A[2][3]

Memory (flat):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│0,0 │0,1 │0,2 │0,3 │1,0 │1,1 │1,2 │1,3 │2,0 │2,1 │2,2 │2,3 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
 ←── row 0 ──→ ←── row 1 ──→ ←── row 2 ──→
```

Iterating row-by-row is cache-friendly; iterating column-by-column causes a cache miss on every access because adjacent columns are `cols * sizeof(elem)` bytes apart.

> [!IMPORTANT]
> Transposing a loop's iteration order (row-major vs. column-major access) can change runtime by 5–10× on large matrices — not because the algorithm changed, but because cache miss rate changes. NumPy's default C order means `arr[i, :]` is fast; `arr[:, j]` is not.

### Python lists vs raw arrays

Python's `list` is **not** a primitive array. Internally it is a `PyListObject` containing a pointer to a heap-allocated array of `PyObject*` pointers (8 bytes each on 64-bit), each pointing to an independently heap-allocated Python object. The memory layout looks like this:

```
PyListObject
┌──────────────────────────────────┐
│ ob_refcnt                        │  8 bytes — reference count
│ ob_type  → &PyList_Type          │  8 bytes — type pointer
│ ob_size  (length)                │  8 bytes — current length
│ ob_item  ────────────────────────┼──► [ *PyObject, *PyObject, *PyObject, … ]
│ allocated (capacity)             │  8 bytes — allocated slots          │
└──────────────────────────────────┘                                     │
                                         ┌───────────────────────────────┘
                                         ▼
                                    ┌────────┬────────┬────────┐
                                    │ ptr[0] │ ptr[1] │ ptr[2] │  ← 8-byte pointers
                                    └────┬───┴────┬───┴────┬───┘
                                         │        │        │
                                    PyIntObject  ...      ...   ← actual ints, elsewhere in heap
```

This means two levels of indirection on every element access, and poor spatial locality: the integers themselves are scattered across the heap. For numerical work this is disastrous — use `numpy.ndarray`, which stores elements contiguously in a true C array.

> [!NOTE]
> CPython's list growth factor is approximately 1.125 (9/8) for small lists, growing faster toward 2× for large lists — not exactly 2× as often stated. The actual formula in `Objects/listobject.c` is `new_allocated = (size_t)newsize + (newsize >> 3) + (newsize < 9 ? 3 : 6)`. The notes' "doubles" description is a simplification.

### Linear search

Linear search is the baseline: scan every element, return the index when a match is found, return -1 if the scan completes without a match. It requires no precondition on the array (unsorted is fine). Time complexity is O(n) worst case; space is O(1).

```python
def linear_search(arr: list[int], target: int) -> int:
    """Return index of target in arr, or -1 if not found. O(n) time, O(1) space."""
    for i, val in enumerate(arr):
        if val == target:
            return i
    return -1
```

An insertion at position `pos` requires shifting all elements from `pos` to end one step right — O(n) in the worst case (inserting at index 0). Deletion at `pos` similarly requires shifting all elements from `pos+1` leftward — also O(n).

### Binary search

Binary search works only on a sorted array. It maintains a live interval `[lo, hi]` and at each step computes `mid = lo + (hi - lo) // 2`, then:

- If `arr[mid] == target`: return `mid`.
- If `arr[mid] < target`: the target must be in `(mid, hi]`, so set `lo = mid + 1`.
- If `arr[mid] > target`: the target must be in `[lo, mid)`, so set `hi = mid - 1`.

Terminate when `lo > hi` (target not found). The idiom `mid = lo + (hi - lo) // 2` instead of `(lo + hi) // 2` is canonical in C/Java/Go where integers can overflow — Python has arbitrary-precision integers so overflow is impossible, but the idiom is worth internalizing.

**Figure:** Binary search step trace on `[10, 20, 30, 40, 50]`, target = 40.

```
Initial: lo=0, hi=4
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
  lo          mid       hi

Step 1: mid=2, arr[2]=30 < 40 → lo = mid+1 = 3
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
               lo  mid  hi
                   (lo=hi=mid=3, arr[3]=40 == target → return 3)

Step 2: mid=3, arr[3]=40 == 40 → return 3  ✓
```

```python
def binary_search_iterative(arr: list[int], target: int) -> int:
    """Return index of target in sorted arr, or -1. O(log n) time, O(1) space."""
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1


def binary_search_recursive(
    arr: list[int], target: int, lo: int, hi: int
) -> int:
    """Recursive binary search. Call with lo=0, hi=len(arr)-1."""
    if lo > hi:
        return -1
    mid = lo + (hi - lo) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search_recursive(arr, target, mid + 1, hi)
    else:
        return binary_search_recursive(arr, target, lo, mid - 1)
```

> [!WARNING]
> The condition must be `lo <= hi`, not `lo < hi`. Using `lo < hi` terminates one iteration early and misses the case where the target is the last remaining element (lo == hi). This is the most common binary search bug.

### Binary search variants

Standard binary search returns *an* index of the target. Many problems need the *first* or *last* occurrence, or a position even when the element is absent. The key insight: instead of returning immediately on a hit, record the hit and continue searching the appropriate half.

**Lower bound** (first index where `arr[i] >= target`): when `arr[mid] >= target`, record `mid` as a candidate and go left (`hi = mid - 1`).

**Upper bound** (first index where `arr[i] > target`): when `arr[mid] <= target`, go right (`lo = mid + 1`); otherwise record and go left.

**First occurrence of target**: lower bound, then check if the result equals target.

**Last occurrence of target**: upper bound minus 1, then check.

```python
def lower_bound(arr: list[int], target: int) -> int:
    """First index i such that arr[i] >= target. Returns len(arr) if all < target."""
    lo, hi = 0, len(arr)          # hi = len(arr): open right end
    while lo < hi:                # invariant: answer in [lo, hi]
        mid = lo + (hi - lo) // 2
        if arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid              # arr[mid] >= target: hi is a candidate
    return lo


def upper_bound(arr: list[int], target: int) -> int:
    """First index i such that arr[i] > target. Returns len(arr) if all <= target."""
    lo, hi = 0, len(arr)
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if arr[mid] <= target:
            lo = mid + 1
        else:
            hi = mid
    return lo


def first_occurrence(arr: list[int], target: int) -> int:
    """Return index of first occurrence of target, or -1."""
    idx = lower_bound(arr, target)
    if idx < len(arr) and arr[idx] == target:
        return idx
    return -1


def last_occurrence(arr: list[int], target: int) -> int:
    """Return index of last occurrence of target, or -1."""
    idx = upper_bound(arr, target) - 1
    if idx >= 0 and arr[idx] == target:
        return idx
    return -1
```

> [!TIP]
> Python's `bisect` module implements `bisect_left` (= lower bound) and `bisect_right` (= upper bound) in C. Use them in production rather than rolling your own. The idiom `bisect.bisect_left(arr, target)` is idiomatic and fast.

## Math

### Recurrence for binary search

At each step, binary search reduces the problem from size n to size n/2 plus O(1) work (the comparison):

```math
T(n) = T\!\left(\frac{n}{2}\right) + O(1), \quad T(1) = O(1)
```

By the Master theorem (case 2, with a=1, b=2, f(n)=O(1), so log_b(a)=0 = degree of f(n)):

```math
T(n) = O(\log n)
```

Recursive binary search uses O(log n) stack space. The iterative version uses O(1) space — preferred for very deep arrays.

### Dynamic array amortized append cost

Starting from an empty array, after n appends the total number of element copies performed during all doublings is:

```math
1 + 2 + 4 + \cdots + \frac{n}{2} < n
```

So n appends cause fewer than n copies total, giving amortized O(1) per append even though individual doublings cost O(n).

### Operation complexities

| Operation | Array (fixed) | Dynamic array | Sorted array |
| :--- | :---: | :---: | :---: |
| Access by index | O(1) | O(1) | O(1) |
| Search (unsorted) | O(n) | O(n) | O(log n) |
| Insert at end | — | O(1) amortized | O(n) (maintain order) |
| Insert at index i | O(n) | O(n) | O(n) |
| Delete at index i | O(n) | O(n) | O(n) |
| Space | O(n) | O(n) (≤2× waste) | O(n) |

## Real-world example

Consider a log aggregation service that writes timestamped events to a sorted on-disk file (timestamps strictly increasing). At query time, given a time range `[t_start, t_end)`, you need to locate the first and last log line within the range. The file is too large to scan linearly — binary search on the timestamp field is the correct primitive.

```python
from __future__ import annotations
import bisect
from dataclasses import dataclass


@dataclass(frozen=True)
class LogEntry:
    timestamp: int   # Unix seconds
    message: str


def find_log_range(
    logs: list[LogEntry],
    t_start: int,
    t_end: int,
) -> list[LogEntry]:
    """
    Return all log entries with timestamp in [t_start, t_end).
    logs must be sorted ascending by timestamp.
    Time: O(log n + k) where k = number of results.
    Space: O(k) for the output slice.
    """
    # bisect_left on a list of objects: extract timestamps into a key list.
    # In production, keep a parallel list[int] of timestamps for O(log n) bisect.
    timestamps: list[int] = [e.timestamp for e in logs]  # O(n) build once

    lo = bisect.bisect_left(timestamps, t_start)   # first index >= t_start
    hi = bisect.bisect_left(timestamps, t_end)     # first index >= t_end (exclusive)
    return logs[lo:hi]


# --- demo ---
sample_logs = [
    LogEntry(1_000, "boot"),
    LogEntry(1_100, "connection accepted"),
    LogEntry(1_200, "request received"),
    LogEntry(1_350, "response sent"),
    LogEntry(1_500, "connection closed"),
]

results = find_log_range(sample_logs, t_start=1_100, t_end=1_400)
for entry in results:
    print(f"  t={entry.timestamp}: {entry.message}")
# t=1100: connection accepted
# t=1200: request received
# t=1350: response sent
```

> [!TIP]
> In production, keep the timestamp list as a separate `array.array('q', ...)` (C long long) rather than a Python list of ints. This gives true contiguous int64 storage and cuts bisect's constant factor because no `PyObject*` dereferencing is needed per comparison.

## In practice

**When binary search loses to a hash table.** Binary search gives O(log n) per query on sorted data. A hash table gives O(1) average per query on unsorted data. For repeated point-queries (not range queries), the hash wins once n is large enough that log n > the hash table's constant factor (empirically around n ≈ 50–200 elements). Use binary search when: the data is already sorted; you need range queries or rank queries; or you cannot afford the hash table's memory overhead.

**Cache-friendliness at scale.** For n ≤ a few thousand elements, linear search often beats binary search due to SIMD auto-vectorization: the compiler unrolls the loop and compares 8–16 elements per instruction. Binary search's log n comparisons each touch a different cache line. The crossover point is hardware-dependent; benchmark before assuming binary search is always faster.

**NumPy for true contiguity.** CPython `list` has pointer indirection; `numpy.ndarray` with `dtype=np.int32` stores 4-byte integers contiguously. Use `np.searchsorted` for binary search on NumPy arrays — it calls into C directly and respects SIMD instructions.

> [!WARNING]
> `np.searchsorted` assumes the array is sorted in ascending order and does **not** verify this. Calling it on an unsorted array returns a meaningless result with no error. Always assert `np.all(arr[:-1] <= arr[1:])` in debug builds if the sort invariant is not guaranteed by construction.

**Dynamic array reallocation jitter.** In latency-sensitive code (game loops, HFT), the O(n) reallocation spike of a dynamic array can cause tail latency outliers. Pre-allocate with `list` constructor: `arr = [None] * expected_capacity` avoids all reallocations.

**2D column iteration penalty.** Accessing `arr[i][j]` in column-major order on a row-major array causes n cache misses for n rows. For large matrices (say 4096×4096 float32 = 64 MB), column iteration is ~10× slower than row iteration on a machine with 64-byte cache lines and 8 MB L3.

## Pitfalls

- **`mid = (lo + hi) // 2` causes integer overflow in C/Java/Go.** In Python, integers are arbitrary-precision and cannot overflow, so this never crashes. But write `lo + (hi - lo) // 2` anyway — it's the portable idiom and interviewers notice.
- **Using `lo < hi` instead of `lo <= hi` as the loop condition.** This misses the case where the target is the sole remaining element (`lo == hi`). The loop exits one step early, returning -1 spuriously.
- **Applying binary search to an unsorted array.** The algorithm's correctness proof relies on the sorted invariant; on an unsorted array, it silently produces wrong answers.
- **Off-by-one in lower/upper bound variants.** The open-right-end convention (`hi = len(arr)`, loop `while lo < hi`) is safer than closed-end (`hi = len(arr)-1`, loop `while lo <= hi`) for lower/upper bound because the "not found" return value (lo = len(arr)) is unambiguous. Mixing conventions in the same codebase is a footgun.
- **Python `list` as a substitute for a true array.** For numerical computation, `list[int]` involves 2 pointer dereferences per element, no SIMD, and 28 bytes of `PyObject` overhead per integer. Use `numpy.ndarray`, `array.array`, or a C extension for anything performance-sensitive.
- **Thinking amortized O(1) append means every append is O(1).** It means the *average* over all appends is O(1). Individual appends at a doubling boundary are O(n). This distinction matters for real-time systems.

## Exercises

The exercises below come from the source notes. Each shows the approach from the PDF, then a clean Python solution with type hints.

### Exercise 1 — Search in a rotated sorted array

A sorted array has been rotated at an unknown pivot. Given the rotated array and a target, return the index or -1. The key observation: one half of the array is always sorted; binary search on that half to decide which way to go.

> [!NOTE]
> This is LeetCode 33. The trick is identifying the sorted half at each step using the relationship between `arr[lo]` and `arr[mid]`.

```python
def search_rotated(arr: list[int], target: int) -> int:
    """
    Binary search on a rotated sorted array. O(log n) time, O(1) space.
    All elements are distinct.
    """
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if arr[mid] == target:
            return mid
        # Left half is sorted
        if arr[lo] <= arr[mid]:
            if arr[lo] <= target < arr[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        # Right half is sorted
        else:
            if arr[mid] < target <= arr[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1


assert search_rotated([4, 5, 6, 7, 0, 1, 2], 0) == 4
assert search_rotated([4, 5, 6, 7, 0, 1, 2], 3) == -1
```

### Exercise 2 — Find peak element

A peak element is strictly greater than its neighbors. Find any peak element in O(log n). The invariant: if `arr[mid] < arr[mid+1]`, a peak must exist in the right half (the right side is going up); otherwise a peak exists in the left half or at mid.

```python
def find_peak(arr: list[int]) -> int:
    """
    Return index of any peak element. O(log n) time, O(1) space.
    Boundary elements are peaks if greater than their single neighbor.
    """
    lo, hi = 0, len(arr) - 1
    while lo < hi:
        mid = lo + (hi - lo) // 2
        if arr[mid] < arr[mid + 1]:
            lo = mid + 1      # right side ascending; peak is right
        else:
            hi = mid          # arr[mid] >= arr[mid+1]; peak is left or at mid
    return lo


assert arr := [1, 2, 3, 1]; find_peak(arr) == 2
assert arr := [1, 2, 1, 3, 5, 6, 4]; find_peak(arr) in {1, 5}
```

### Exercise 3 — Integer square root via binary search

Find the floor of the square root of a non-negative integer x without using `math.sqrt`. Binary search on the answer space [0, x]: find the largest integer m such that m*m <= x.

```python
def isqrt_bsearch(x: int) -> int:
    """
    Return floor(sqrt(x)) using binary search. O(log x) time, O(1) space.
    Equivalent to math.isqrt(x) but illustrates binary search on answer space.
    """
    if x < 2:
        return x
    lo, hi = 1, x // 2
    result = 0
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        sq = mid * mid
        if sq == x:
            return mid
        elif sq < x:
            result = mid
            lo = mid + 1
        else:
            hi = mid - 1
    return result


for n in [0, 1, 4, 8, 9, 10, 25, 26]:
    import math
    assert isqrt_bsearch(n) == math.isqrt(n), f"failed for {n}"
```

### Exercise 4 — First/last occurrence in a sorted array with duplicates

Given a sorted array that may contain duplicates, return the first and last index of a target value.

```python
def first_last_occurrence(arr: list[int], target: int) -> tuple[int, int]:
    """
    Return (first_index, last_index) of target, or (-1, -1) if absent.
    O(log n) time, O(1) space.
    """
    lo_idx = lower_bound(arr, target)
    if lo_idx == len(arr) or arr[lo_idx] != target:
        return (-1, -1)
    hi_idx = upper_bound(arr, target) - 1
    return (lo_idx, hi_idx)


assert first_last_occurrence([1, 2, 2, 2, 3, 4], 2) == (1, 3)
assert first_last_occurrence([1, 2, 3], 5) == (-1, -1)
```

### Exercise 5 — Maximum subarray sum (Kadane's algorithm)

Find the contiguous subarray with the largest sum. At each index, either extend the running subarray or start fresh — whichever gives the larger result.

```python
def max_subarray_sum(arr: list[int]) -> int:
    """
    Return the maximum sum of any non-empty contiguous subarray.
    Kadane's algorithm: O(n) time, O(1) space.
    """
    max_sum = arr[0]
    cur_sum = arr[0]
    for val in arr[1:]:
        cur_sum = max(val, cur_sum + val)
        max_sum = max(max_sum, cur_sum)
    return max_sum


assert max_subarray_sum([-5, 1, 2, 3, -1, 2, -3]) == 7   # [1,2,3,-1,2]
assert max_subarray_sum([-2, -3, 4, -1, -2, 1, 5, -3]) == 7
```

### Exercise 6 — Majority element (Boyer-Moore voting)

Find the element that appears more than n//2 times. Phase 1: walk the array maintaining a (candidate, count) pair — increment if current element equals candidate, decrement otherwise; reset candidate when count hits 0. Phase 2: verify the candidate (optional if the problem guarantees a majority exists).

```python
def majority_element(arr: list[int]) -> int | None:
    """
    Return element appearing > n//2 times, or None if no majority exists.
    Boyer-Moore voting: O(n) time, O(1) space.
    """
    candidate = arr[0]
    count = 1
    for val in arr[1:]:
        if count == 0:
            candidate = val
            count = 1
        elif val == candidate:
            count += 1
        else:
            count -= 1
    # Phase 2: verify (required when majority not guaranteed)
    if arr.count(candidate) > len(arr) // 2:
        return candidate
    return None


assert majority_element([3, 3, 4, 2, 3, 3, 3]) == 3
assert majority_element([1, 2, 3, 4]) is None
```

### Exercise 7 — Maximum difference (buy low, sell high)

Given stock prices over n days, find the maximum profit from one buy-sell transaction (buy before sell). Track the minimum price seen so far and update max profit at each step.

```python
def max_profit_single(prices: list[int]) -> int:
    """
    Maximum profit from one buy-sell transaction. O(n) time, O(1) space.
    Returns 0 if no profitable transaction exists.
    """
    if len(prices) < 2:
        return 0
    min_price = prices[0]
    max_profit = 0
    for price in prices[1:]:
        max_profit = max(max_profit, price - min_price)
        min_price = min(min_price, price)
    return max_profit


assert max_profit_single([7, 1, 5, 3, 6, 4]) == 5   # buy at 1, sell at 6
assert max_profit_single([7, 6, 4, 3, 1]) == 0       # no profitable trade
```

### Exercise 8 — Prefix sum and equilibrium index

An equilibrium index is an index i where the sum of elements to the left equals the sum to the right. Precompute prefix sums, then for each index check left_sum == total - left_sum - arr[i].

```python
def find_equilibrium(arr: list[int]) -> int:
    """
    Return the first equilibrium index, or -1.
    O(n) time (one prefix-sum pass + one scan), O(1) extra space.
    """
    total = sum(arr)
    left_sum = 0
    for i, val in enumerate(arr):
        # right_sum = total - left_sum - val
        if left_sum == total - left_sum - val:
            return i
        left_sum += val
    return -1


assert find_equilibrium([3, 1, 8, 2, 6]) == 2   # left=[3,1], right=[2,6]
assert find_equilibrium([1, 2, 3]) == -1
```

## Sources

- Handwritten lecture notes: "Array + Searching", scanned PDF, course on Data Structures and Algorithms (2024).
- CPython source: `Objects/listobject.c` — `list_resize()` for the dynamic growth formula. https://github.com/python/cpython/blob/main/Objects/listobject.c
- Python `bisect` module documentation. https://docs.python.org/3/library/bisect.html
- Cormen, T. H., Leiserson, C. E., Rivest, R. L., & Stein, C. (2022). *Introduction to Algorithms*, 4th ed. MIT Press. §§ 2.1 (insertion sort / array ops), 2.3 (divide-and-conquer / binary search).
- Kadane, J. B. (1984). Maximum sum contiguous subarray problem. *Communications of the ACM*.
- Boyer, R. S., & Moore, J. S. (1991). MJRTY — A Fast Majority Vote Algorithm. *Automated Reasoning*, 105–117.
- Conversation with user on 2026-05-19.

## Related

- [[01-intro-and-recursion]]
- [[03-sorting]]
- [[04-hashing]]
