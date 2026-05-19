---
title: "Sorting"
created: 2026-05-19
updated: 2026-05-19
tags: [data-structures-and-algorithms, sorting, algorithms, comparison-sort, merge-sort, quicksort, timsort]
aliases: ["Sorting Algorithms", "03-sorting"]
---
# Sorting

[toc]

> **TL;DR:** Sorting rearranges a sequence into a defined order and is the foundation of countless algorithms that exploit sorted structure (binary search, merge-join, duplicate detection). Comparison-based sorts hit a hard theoretical floor of Ω(n log n) comparisons; non-comparison sorts (counting, radix, bucket) break the floor by exploiting value-domain structure. Knowing *which* algorithm to pick — stable vs. unstable, in-place vs. buffered, adaptive vs. oblivious — is as important as knowing how each one works.

## Vocabulary

**Stable sort** — a sort that preserves the relative order of elements with equal keys. If `a[i] == a[j]` and `i < j` before sorting, then the record originally at `i` appears before the record originally at `j` in the output. Bubble sort, insertion sort, and merge sort are stable; selection sort, quicksort (both partitions), and heapsort are not.

**In-place sort** — a sort that uses O(1) auxiliary space beyond the input array. Bubble, selection, insertion, and heapsort are in-place; mergesort requires O(n) auxiliary; counting/radix/bucket require O(n + k) auxiliary.

**Adaptive sort** — a sort whose running time improves when the input is already partially sorted. Insertion sort is strongly adaptive: O(n) on a sorted input, O(n + inversions) in general. Mergesort and heapsort are non-adaptive.

**Comparison sort** — a sort that determines order solely by pairwise element comparisons. All comparison sorts are bounded below by Ω(n log n) in the worst case.

**Non-comparison sort** — exploits the structure of keys (integer range, digit count) to sort without element comparisons. Can achieve O(n) time under the right assumptions.

**Pivot** — the element chosen in quicksort around which the array is partitioned. Its final position after partitioning is its correct sorted position.

**Partition** — the operation that rearranges an array so all elements ≤ pivot appear before the pivot and all elements > pivot appear after it (Lomuto convention). The pivot lands in its final sorted position.

**Run** — a maximal contiguous subsequence that is already sorted. Timsort detects existing runs and exploits them; a fully sorted input has one run.

**Decision tree** — a full binary tree where each internal node is a comparison `a[i] : a[j]`, each leaf is a permutation (one possible sorted order), and a path from root to leaf is one execution of the algorithm. The height of the tree is a lower bound on worst-case comparisons.

**Inversion** — a pair (i, j) with i < j and a[i] > a[j]. The number of inversions measures "how unsorted" an array is; insertion sort performs exactly one swap per inversion.

**Timsort** — Python's and Java's production sort. A hybrid of natural mergesort and binary insertion sort. Detects existing ascending or descending runs, reverses descending runs in place, extends short runs with binary insertion sort to a minimum run length (minrun, 32–64), then merges runs with a galloping merge that exploits long monotone streaks.

**Lomuto partition** — quicksort partition scheme that uses the last element as pivot, maintains a single boundary `i` (elements ≤ pivot are left of `i`), and scans left-to-right with one pointer. Simpler code; O(n²) on sorted input if pivot selection is naive.

**Hoare partition** — original partition scheme: two pointers converge from both ends, swapping elements out of place. Fewer swaps than Lomuto in practice; the final pivot position is not `i` after the call — the recursion must be `[l, i]` + `[i+1, r]`.

## Intuition

Imagine a deck of cards you need to sort by rank. Your first instinct is to scan for the smallest card and pull it out — that is selection sort. Or you might pick cards one at a time and slot each into its correct position in a growing sorted hand — that is insertion sort. Both work but both require O(n²) comparisons in the worst case because you might compare every card against every other.

The key insight behind merge sort and quicksort is *divide-and-conquer*: split the problem, solve the halves independently, and combine. Each level of splitting costs O(n) work; there are O(log n) levels; total cost O(n log n). The n log n floor is not arbitrary — it is the minimum number of binary questions needed to distinguish among n! possible input orderings, since a binary tree with n! leaves must have height at least log₂(n!), and by Stirling's approximation log₂(n!) ≈ n log₂ n.

Non-comparison sorts sidestep this argument entirely. If you know your keys are integers in [0, k), you can sort by counting occurrences rather than by comparing — O(n + k) regardless of n log n.

## Math foundations

The n log n barrier for comparison sorts is one of the cleanest lower-bound arguments in all of CS. The five subsections below derive it from first principles, show how merge sort achieves it optimally, characterize quicksort's expected behavior via indicator random variables, and then explain precisely why counting and radix sort are not contradictions of the bound—they simply escape the model the bound applies to.

### Permutations and the comparison decision tree

Any deterministic comparison sort on n elements can be modeled as a **full binary decision tree**: each internal node is a comparison `a[i] : a[j]` with two branches (≤ or >), and each leaf is a complete permutation of the input. Because the algorithm must handle every possible permutation correctly, the tree must have **at least n! leaves**. A binary tree of height h has at most 2^h leaves, giving the fundamental inequality. Every root-to-leaf path is one execution of the algorithm; the longest path is the worst-case comparison count.

```math
2^h \geq n!
\implies h \geq \log_2(n!)
```

Since h is an integer, the worst-case comparison count is at least ⌈log₂(n!)⌉. By Stirling (next subsection), log₂(n!) = Θ(n log n), which establishes the **Ω(n log n) lower bound** for all comparison sorts. This bound is **tight**: merge sort and heapsort both achieve Θ(n log n) worst-case.

> [!IMPORTANT]
> This lower bound applies only to algorithms that determine order *exclusively* through pairwise comparisons. It says nothing about algorithms that read key bits directly. Counting sort and radix sort are outside this model — they are not comparison sorts.

### Stirling's approximation

Stirling's formula provides a tight asymptotic estimate for n! that makes the decision-tree height explicitly computable. The direct factorial is expensive for large n; Stirling replaces it with a closed form tight to within a multiplicative factor of 1 + O(1/n). The key consequence for sorting is that log₂(n!) = Θ(n log n), establishing both the upper and lower constants of the lower bound.

```math
n! \approx \sqrt{2\pi n} \left(\frac{n}{e}\right)^n
```

Taking the natural logarithm of both sides:

```math
\ln(n!) \approx \frac{1}{2}\ln(2\pi n) + n\ln n - n = n\ln n - n + O(\ln n)
```

Converting to log base 2 by dividing by ln 2:

```math
\log_2(n!) = \frac{\ln(n!)}{\ln 2} = \frac{n\ln n - n + O(\ln n)}{\ln 2} = \Theta(n \log_2 n)
```

A more elementary lower bound uses the integral approximation of the log-sum:

```math
\ln(n!) = \sum_{k=1}^{n} \ln k \geq \int_{1}^{n} \ln x \, dx = \bigl[x \ln x - x\bigr]_1^n = n\ln n - n + 1
```

Both approaches confirm log₂(n!) = Θ(n log n), completing the lower-bound proof for the decision-tree argument.

### The Master theorem recurrence for merge sort

Merge sort is the canonical algorithm that *achieves* the Θ(n log n) lower bound. Its divide step splits at the midpoint (O(1) work), recurses on two halves of size n/2, and combines with a linear-time two-pointer merge. This is a textbook application of Master theorem Case 2, where the driving function f(n) = Θ(n) exactly matches n^(log_b a).

```math
T(n) = 2\,T\!\left(\frac{n}{2}\right) + \Theta(n)
```

Applying the Master theorem with a = 2, b = 2, f(n) = Θ(n):

```math
n^{\log_b a} = n^{\log_2 2} = n^1 = \Theta(n) = \Theta(f(n))
\implies T(n) = \Theta(n \log n)
```

The recursion tree makes the accounting concrete: the tree has log₂ n levels, and every level performs exactly Θ(n) total work (the merge step spans all n elements at that level). The total is their product.

```
Level 0:      [─────────────── n ───────────────]            work = Θ(n)
Level 1:      [────── n/2 ──────][────── n/2 ──────]         work = 2×Θ(n/2) = Θ(n)
Level 2:      [──n/4──][──n/4──][──n/4──][──n/4──]           work = 4×Θ(n/4) = Θ(n)
  ...
Level k:      2^k subproblems of size n/2^k                   work = Θ(n)
  ...
Level log₂n:  n subproblems of size 1                         work = Θ(n)

Total levels = log₂ n + 1
Total work   = Θ(n log n)
```

> [!NOTE]
> The per-level work is exactly Θ(n) because the two-pointer merge is linear. If the merge were superlinear — for example O(n log n) per level, as in a naive implementation using an internal comparison sort — the recurrence would yield T(n) = Θ(n log² n). The linearity of the merge is load-bearing for the Θ(n log n) result.

### Expected-time analysis for randomized quicksort

Randomized quicksort picks the pivot uniformly at random from the current subarray. The expected comparison count is analyzed with the **indicator random variable** technique (CLRS §7.4): define X_ij = 1 if elements a_i and a_j (indices in sorted order, i < j) are ever compared directly, and 0 otherwise. The total comparison count is the sum of all X_ij. Two elements are compared exactly when one of them is the first pivot chosen from the contiguous range {a_i, a_{i+1}, ..., a_j} — any other element chosen first splits them before they can meet. Among the j − i + 1 elements in that range, each is equally likely to be chosen first.

```math
\Pr[a_i \text{ compared to } a_j] = \frac{2}{j - i + 1}
```

Summing over all pairs (i < j) with the substitution d = j − i:

```math
E[\text{total comparisons}]
  = \sum_{i=1}^{n-1} \sum_{j=i+1}^{n} \frac{2}{j - i + 1}
  = 2 \sum_{i=1}^{n-1} \sum_{d=1}^{n-i} \frac{1}{d+1}
  \leq 2n \sum_{d=1}^{n} \frac{1}{d}
  = 2n H_n
```

where H_n is the n-th harmonic number:

```math
H_n = \sum_{k=1}^{n} \frac{1}{k} \approx \ln n + \gamma, \quad \gamma \approx 0.5772
```

Therefore:

```math
E[\text{total comparisons}] \leq 2n H_n \approx 2n \ln n = \Theta(n \log n)
```

> [!TIP]
> The indicator RV technique avoids tracking the full partition-tree structure. Exactly the same argument gives O(n) expected comparisons for quickselect, O(1) expected chain length per slot in a uniform hash table, and O(log n) expected level height in a randomized skip list.

### Counting and radix sort break the lower bound

Counting and radix sort are not comparison sorts — they never evaluate `a[i] < a[j]` and branch on the result. They read the *bit representation* of keys directly, placing them outside the decision-tree model the lower bound analyzes. There is no contradiction: the Ω(n log n) bound is a statement about comparison-based algorithms only. Counting sort uses zero element-to-element comparisons.

**Counting sort** builds a frequency histogram over keys in [0, k), computes prefix sums to determine each key's output position, and places elements in a single backward pass for stability. The non-negative integer keys in a bounded range assumption is load-bearing.

```math
T_{\text{counting}}(n, k) = O(n + k)
```

This is o(n log n) when k = O(n) (e.g., sorting bytes). It is impractical when k ≫ n: sorting 1000 elements with 32-bit keys would need a 2^32-entry count array.

**Radix sort** applies a stable counting sort once per digit position from least to most significant (LSD radix sort), with base b and d digit positions:

```math
T_{\text{radix}}(n, b, d) = O\!\left(d\,(n + b)\right)
```

For 32-bit integer keys in base 256 (b = 256, d = 4):

```math
T = O\!\left(4\,(n + 256)\right) = O(n)
```

The break-even against Θ(n log n) comparison sort solves d(n + b) = n log₂ n. For base-256 32-bit keys: 4(n + 256) = n log₂ n, giving a crossover at n ≈ 1000. Below ~1000 elements, cache effects and small constants keep comparison sorts competitive.

> [!WARNING]
> The O(n + k) bound for counting sort silently assumes non-negative integer keys in [0, k). Negative integers, floating-point values, or variable-length strings require a key-mapping step. Radix sort handles signed integers by flipping the sign bit on the final (most-significant) digit pass, but standard implementations omit this and are correct for non-negative values only.


## Comparison Table

| Algorithm | Best | Average | Worst | Space | Stable | In-place | Adaptive |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Bubble sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Yes |
| Selection sort | O(n²) | O(n²) | O(n²) | O(1) | No | Yes | No |
| Insertion sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Yes | Yes |
| Merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | No | No |
| Quicksort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Yes* | No |
| Heap sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Yes | No |
| Counting sort | O(n + k) | O(n + k) | O(n + k) | O(k) | Yes | No | No |
| Radix sort | O(nw) | O(nw) | O(nw) | O(n + k) | Yes | No | No |
| Bucket sort | O(n + k) | O(n + k) | O(n²) | O(n + k) | Yes† | No | No |
| Timsort | O(n) | O(n log n) | O(n log n) | O(n) | Yes | No | Yes |

*Quicksort stack space O(log n) average; O(n) worst without tail-call optimization. †Bucket sort stability depends on the inner sort used per bucket.

## How it Works

Each algorithm below solves the same problem — rearranging an unsorted array into ascending order — but makes different trade-offs in comparisons, swaps, memory, and behavior on structured inputs. The ASCII step-traces show the array contents after each pass so you can see exactly what memory holds at each point in time.

### Bubble Sort

Bubble sort makes repeated passes over the array. On each pass it compares adjacent pairs and swaps them if they are out of order. The largest unsorted element "bubbles" to its correct position at the end of each pass. After k passes, the last k elements are in their final positions, so the inner loop can shrink by one each time. An early-exit optimization: if a full pass completes with no swaps, the array is already sorted and the algorithm stops — this gives O(n) best-case on sorted input.

```
Input:  [5, 3, 8, 1, 2]

Pass 1: [3, 5, 1, 2, 8]   ← 8 bubbles to end (4 comparisons, 3 swaps)
Pass 2: [3, 1, 2, 5, 8]   ← 5 bubbles to position 3
Pass 3: [1, 2, 3, 5, 8]   ← 3 bubbles to position 2
Pass 4: [1, 2, 3, 5, 8]   ← no swaps → early exit
```

```python
from typing import List


def bubble_sort(arr: List[int]) -> None:
    """Sort arr in-place ascending. Stable. O(n²) worst, O(n) best."""
    n = len(arr)
    for i in range(n - 1):
        swapped = False
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True
        if not swapped:
            break  # already sorted
```

### Selection Sort

Selection sort divides the array into a sorted prefix and an unsorted suffix. On each pass it scans the unsorted suffix for the minimum element and swaps it into the first position of the suffix. Unlike bubble sort, the number of comparisons is always n(n-1)/2 regardless of input order — it is completely non-adaptive. Crucially, the swap-into-position step can disturb the relative order of equal elements, making selection sort **unstable**.

```
Input:    [5, 3, 8, 1, 2]
           ^sorted |^ unsorted^

After i=0: [1, 3, 8, 5, 2]   ← min=1 swapped to front
After i=1: [1, 2, 8, 5, 3]   ← min=2 swapped to pos 1
After i=2: [1, 2, 3, 5, 8]   ← min=3 swapped to pos 2
After i=3: [1, 2, 3, 5, 8]   ← min=5 already in place
```

```python
from typing import List


def selection_sort(arr: List[int]) -> None:
    """Sort arr in-place ascending. Unstable. O(n²) always."""
    n = len(arr)
    for i in range(n - 1):
        min_idx = i
        for j in range(i + 1, n):
            if arr[j] < arr[min_idx]:
                min_idx = j
        arr[i], arr[min_idx] = arr[min_idx], arr[i]
```

### Insertion Sort

Insertion sort maintains a sorted prefix by taking the next unsorted element (the "key") and shifting all sorted elements that are larger than the key one position to the right, then placing the key in the resulting gap. This is exactly how you sort a hand of playing cards. The number of shifts equals the number of inversions, so the algorithm is strongly adaptive: O(n) on a sorted input, O(n + k) where k is the number of inversions. It is stable because equal elements are never moved past each other.

```python
Input:         [50, 20, 40, 60, 10]

i=1, key=20:   [50, 50, 40, 60, 10]  → shift 50 right
               [20, 50, 40, 60, 10]  → place key

i=2, key=40:   [20, 50, 50, 60, 10]  → shift 50 right
               [20, 40, 50, 60, 10]  → place key

i=3, key=60:   [20, 40, 50, 60, 10]  → 60 already in place

i=4, key=10:   [20, 40, 50, 60, 60]  → shift 60
               [20, 40, 50, 50, 60]  → shift 50
               [20, 40, 40, 50, 60]  → shift 40
               [20, 20, 40, 50, 60]  → shift 20
               [10, 20, 40, 50, 60]  → place key
```

```python
from typing import List


def insertion_sort(arr: List[int]) -> None:
    """Sort arr in-place ascending. Stable. O(n) best, O(n²) worst."""
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        # shift elements greater than key one position to the right
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
```

> [!NOTE]
> Timsort and C++ introsort both fall back to insertion sort for subarrays below a threshold (typically 16–64 elements). On small arrays the low constant factor of insertion sort outperforms the O(n log n) sorts because cache effects dominate.

### Merge Sort

Merge sort is a divide-and-conquer algorithm. It recursively splits the array at the midpoint, sorts each half, and merges the two sorted halves into a single sorted array. The merge step is the heart of the algorithm: two pointers scan the two halves and always copy the smaller element, taking O(n) time and O(n) auxiliary space. The key invariant that guarantees stability is to prefer the left-half element when `left[i] == right[j]` — the `<=` in the comparison, not `<`.

```
Input: [10, 5, 30, 15, 7]
                                    split
            [10, 5, 30]                    [15, 7]
           /           \                  /      \
       [10, 5]        [30]           [15]        [7]
       /     \
    [10]     [5]
              ↑ base cases
    merge ↑
    [5, 10]       +      [30]     →   [5, 10, 30]
                         merge ↑
                    [7, 15]
                         merge ↑
    [5, 10, 30]    +    [7, 15]  →   [5, 7, 10, 15, 30]
```

The merge function for the merge-sort algorithm (operating on a subarray `arr[l..r]` with midpoint `m`):

```
Memory layout during merge of arr[l..r], midpoint m:
┌──────────────────────────────────────────┐
│  arr:  [ l ... m ][ m+1 ... r ]          │
│         ↑ sorted    ↑ sorted             │
│         copy to left[]  copy to right[]  │
└──────────────────────────────────────────┘

left[]  ← arr[l .. m]       (n1 = m - l + 1 elements)
right[] ← arr[m+1 .. r]     (n2 = r - m elements)

Two-pointer merge back into arr[l..r]:
  i → left,  j → right,  k → arr starting at l
  while i < n1 and j < n2:
      if left[i] <= right[j]:  arr[k++] = left[i++]  ← <= preserves stability
      else:                     arr[k++] = right[j++]
  drain remaining left / right into arr
```

```python
from typing import List


def merge_sort(arr: List[int], l: int, r: int) -> None:
    """Sort arr[l..r] in-place (uses O(n) auxiliary). Stable. O(n log n)."""
    if r <= l:
        return
    m = l + (r - l) // 2  # avoids integer overflow vs (l + r) // 2
    merge_sort(arr, l, m)
    merge_sort(arr, m + 1, r)
    _merge(arr, l, m, r)


def _merge(arr: List[int], l: int, m: int, r: int) -> None:
    """Merge arr[l..m] and arr[m+1..r] into arr[l..r]."""
    n1 = m - l + 1
    n2 = r - m
    left = arr[l : l + n1]    # copy left half
    right = arr[m + 1 : r + 1]  # copy right half
    i = j = 0
    k = l
    while i < n1 and j < n2:
        if left[i] <= right[j]:   # <= is the stability invariant
            arr[k] = left[i]
            i += 1
        else:
            arr[k] = right[j]
            j += 1
        k += 1
    while i < n1:
        arr[k] = left[i]
        i += 1
        k += 1
    while j < n2:
        arr[k] = right[j]
        j += 1
        k += 1
```

> [!IMPORTANT]
> Use `m = l + (r - l) // 2` rather than `(l + r) // 2`. If `l` and `r` are large integers (e.g. indices into a very large array in a language with fixed-width ints), their sum can overflow. Python ints are arbitrary-precision so overflow is not an issue in Python, but the habit carries to C/Java/Go.

### Quicksort — Lomuto Partition

Quicksort picks a pivot, partitions the array so all elements ≤ pivot appear to its left and all elements > pivot appear to its right, then recursively sorts both sides. The Lomuto scheme uses the **last element** as the pivot and maintains a single boundary index `i`: everything at or before `i` is ≤ pivot. It scans left to right with `j`; when `arr[j] <= pivot` it increments `i` and swaps `arr[i]` with `arr[j]`. At the end, the pivot swaps into position `i + 1`.

```
Input: [3, 8, 6, 12, 10, 7]   pivot = arr[5] = 7

j=0: arr[0]=3 ≤ 7  → i=0, swap(arr[0],arr[0])  → [3, 8, 6, 12, 10, 7]
j=1: arr[1]=8 > 7  → skip
j=2: arr[2]=6 ≤ 7  → i=1, swap(arr[1],arr[2])  → [3, 6, 8, 12, 10, 7]
j=3: arr[3]=12 > 7 → skip
j=4: arr[4]=10 > 7 → skip
end: swap pivot arr[5] with arr[i+1]=arr[2]     → [3, 6, 7, 12, 10, 8]
                                                            ↑
                                                   pivot in final position
Recurse on [3, 6] and [12, 10, 8]
```

```python
from typing import List


def quicksort_lomuto(arr: List[int], l: int, r: int) -> None:
    """Quicksort using Lomuto partition. Unstable. O(n log n) avg, O(n²) worst."""
    if l < r:
        p = _partition_lomuto(arr, l, r)
        quicksort_lomuto(arr, l, p - 1)
        quicksort_lomuto(arr, p + 1, r)


def _partition_lomuto(arr: List[int], l: int, r: int) -> int:
    """Partition arr[l..r] around arr[r]. Returns final pivot index."""
    pivot = arr[r]
    i = l - 1
    for j in range(l, r):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1], arr[r] = arr[r], arr[i + 1]
    return i + 1
```

### Quicksort — Hoare Partition

The Hoare partition scheme is the original: two pointers `i` and `j` start at the left and right ends and march toward each other, swapping pairs that are on the wrong side. It stops when `i >= j`. The pivot is typically placed at `arr[l]`. Hoare's scheme does about 3× fewer swaps than Lomuto on random data. The critical difference: after `_partition_hoare` returns index `p`, the pivot is **not** guaranteed to be at position `p`; the recursion must cover `[l, p]` and `[p+1, r]`, not `[l, p-1]` and `[p+1, r]`.

```
Input: [3, 8, 6, 12, 10, 7]   pivot = arr[0] = 3
i=0, j=5

i advances while arr[i] < 3: stops at i=0 (arr[0]=3 not < 3)
j retreats while arr[j] > 3: stops at j=0 (arr[0]=3 not > 3)
i >= j → stop. Return j=0.

Recurse on [l=0, p=0] (single element) and [p+1=1, r=5]
```

```python
from typing import List


def quicksort_hoare(arr: List[int], l: int, r: int) -> None:
    """Quicksort using Hoare partition. Unstable. O(n log n) avg, O(n²) worst."""
    if l < r:
        p = _partition_hoare(arr, l, r)
        quicksort_hoare(arr, l, p)       # note: p, not p-1
        quicksort_hoare(arr, p + 1, r)


def _partition_hoare(arr: List[int], l: int, r: int) -> int:
    """Hoare partition around arr[l]. Returns split index (not pivot position)."""
    pivot = arr[l]
    i = l - 1
    j = r + 1
    while True:
        i += 1
        while arr[i] < pivot:
            i += 1
        j -= 1
        while arr[j] > pivot:
            j -= 1
        if i >= j:
            return j
        arr[i], arr[j] = arr[j], arr[i]
```

> [!WARNING]
> A common off-by-one: with Hoare partition the recursive call is `quicksort(arr, l, p)` and `quicksort(arr, p+1, r)`. If you write `quicksort(arr, l, p-1)` (as you would with Lomuto), you will skip elements and produce an incorrect sort. The bug can be hard to catch because the error only manifests when equal elements or the pivot itself is on the boundary.

### Heap Sort

Heap sort works in two phases. Phase 1 (build-heap): convert the array into a max-heap in O(n) time by calling `heapify` on every internal node from the last one up to the root. Phase 2 (extract-max): repeatedly swap the root (max element) with the last unsorted element, shrink the heap by one, and restore the heap property by calling `heapify` on the root. The algorithm is in-place and O(n log n) in all cases, but it is not stable because the heap operations reorder equal elements. In practice, heapsort is rarely used in production over quicksort because its access pattern is cache-unfriendly (it jumps around the array rather than scanning linearly).

```
Build-max-heap from [4, 10, 3, 5, 1]:

Start from last internal node (idx 1 = parent of leaf at idx 3 or 4):
  heapify at idx 1: children are 10→idx3=5, 4→idx4=1. 10 largest, no swap needed.
  heapify at idx 0: children are arr[1]=10, arr[2]=3. Swap 4↔10:
  → [10, 4, 3, 5, 1]
  heapify at idx 1 again: children arr[3]=5, arr[4]=1. Swap 4↔5:
  → [10, 5, 3, 4, 1]   max-heap built

Extract phase:
  swap root↔last → [1, 5, 3, 4, | 10]  heapify root → [5, 4, 3, 1, | 10]
  swap root↔last → [1, 4, 3, | 5, 10]  heapify root → [4, 1, 3, | 5, 10]
  swap root↔last → [3, 1, | 4, 5, 10]  heapify root → [3, 1, | 4, 5, 10]
  swap root↔last → [1, | 3, 4, 5, 10]  done
Final: [1, 3, 4, 5, 10]
```

```python
from typing import List


def heap_sort(arr: List[int]) -> None:
    """Sort arr in-place ascending. Unstable. O(n log n) always."""
    n = len(arr)
    # Phase 1: build max-heap (start from last internal node)
    for i in range(n // 2 - 1, -1, -1):
        _heapify(arr, n, i)
    # Phase 2: extract max elements one by one
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]   # move current max to end
        _heapify(arr, i, 0)               # restore heap over arr[0..i-1]


def _heapify(arr: List[int], heap_size: int, root: int) -> None:
    """Sift-down: ensure arr[root] satisfies max-heap property."""
    largest = root
    left = 2 * root + 1
    right = 2 * root + 2
    if left < heap_size and arr[left] > arr[largest]:
        largest = left
    if right < heap_size and arr[right] > arr[largest]:
        largest = right
    if largest != root:
        arr[root], arr[largest] = arr[largest], arr[root]
        _heapify(arr, heap_size, largest)
```

### Counting Sort

Counting sort is a non-comparison sort for integer keys in a known range [0, k). It counts how many times each key appears, then computes prefix sums to determine the output position of each element. It is stable if you iterate the input right-to-left when placing elements. Time O(n + k); space O(k) for the count array plus O(n) for the output. It is impractical when k >> n (e.g., sorting 100 elements with 32-bit integer keys).

```
Input: [4, 2, 2, 8, 3, 3, 1]   k = 9 (keys in [0,8])

count:  [0, 1, 2, 2, 1, 0, 0, 0, 1]   (index = value, val = frequency)
prefix: [0, 0, 1, 3, 5, 6, 6, 6, 6]   (prefix sum → start position)

Place elements right-to-left for stability:
  arr[6]=1 → output[prefix[1]=0]=1, prefix[1]++
  arr[5]=3 → output[prefix[3]=3]=3, prefix[3]++
  ...
Output: [1, 2, 2, 3, 3, 4, 8]
```

```python
from typing import List


def counting_sort(arr: List[int], k: int) -> List[int]:
    """
    Sort arr with non-negative integer keys in [0, k).
    Stable. O(n + k) time and space.
    Returns a new sorted list.
    """
    count: List[int] = [0] * k
    for x in arr:
        count[x] += 1
    # prefix sum: count[i] = number of elements < i (= start index of i in output)
    for i in range(1, k):
        count[i] += count[i - 1]
    # place elements right-to-left to maintain stability
    output: List[int] = [0] * len(arr)
    for x in reversed(arr):
        count[x] -= 1
        output[count[x]] = x
    return output
```

### Radix Sort

Radix sort generalises counting sort to multi-digit keys. It sorts by each digit position from least significant to most significant (LSD radix sort), using a stable sort (typically counting sort) as the inner sorter at each position. After w passes (where w is the number of digits in the largest key), the array is sorted. Total time O(w × (n + b)) where b is the base (typically 10 or 256); total space O(n + b).

```
Input: [170, 45, 75, 90, 802, 24, 2, 66]   base 10, w=3

Pass 1 (ones digit):
  170  45  75  90  802  24   2  66
  ↓0   ↓5  ↓5  ↓0  ↓2  ↓4  ↓2 ↓6
  → [170, 90, 802, 2, 24, 45, 75, 66]

Pass 2 (tens digit):
  → [802, 2, 24, 45, 66, 170, 75, 90]

Pass 3 (hundreds digit):
  → [2, 24, 45, 66, 75, 90, 170, 802]
```

```python
from typing import List


def radix_sort(arr: List[int]) -> List[int]:
    """
    LSD radix sort for non-negative integers. Base 10.
    Stable. O(w*(n+10)) time, O(n+10) space where w = digits in max element.
    Returns a new sorted list.
    """
    if not arr:
        return []
    max_val = max(arr)
    exp = 1
    result = list(arr)
    while max_val // exp > 0:
        result = _counting_sort_by_digit(result, exp)
        exp *= 10
    return result


def _counting_sort_by_digit(arr: List[int], exp: int) -> List[int]:
    n = len(arr)
    output: List[int] = [0] * n
    count: List[int] = [0] * 10
    for x in arr:
        digit = (x // exp) % 10
        count[digit] += 1
    for i in range(1, 10):
        count[i] += count[i - 1]
    for x in reversed(arr):
        digit = (x // exp) % 10
        count[digit] -= 1
        output[count[digit]] = x
    return output
```

### Bucket Sort

Bucket sort distributes elements into a fixed number of buckets based on their value range, sorts each bucket individually (typically with insertion sort), then concatenates. It achieves O(n) average time when input is uniformly distributed — each bucket gets O(1) elements on average. Worst case (all elements in one bucket) degrades to O(n²) if the inner sort is O(n²). It is well-suited for floating-point values in [0, 1).

```
Input: [0.78, 0.17, 0.39, 0.26, 0.72, 0.94, 0.21, 0.12]
n=8 buckets (index = floor(n * x)):

Bucket 0: []
Bucket 1: [0.17, 0.12]
Bucket 2: [0.26, 0.21]
Bucket 3: [0.39]
Bucket 4: []
...
Bucket 7: [0.78, 0.72]
Bucket 9: [0.94]   (last bucket = floor(8*0.94)=7, same bucket)

Sort each bucket → concat → [0.12, 0.17, 0.21, 0.26, 0.39, 0.72, 0.78, 0.94]
```

```python
from typing import List


def bucket_sort(arr: List[float]) -> List[float]:
    """
    Sort floats in [0, 1). Stable (insertion sort per bucket).
    O(n) average, O(n²) worst. Returns a new sorted list.
    """
    if not arr:
        return []
    n = len(arr)
    buckets: List[List[float]] = [[] for _ in range(n)]
    for x in arr:
        idx = int(x * n)
        idx = min(idx, n - 1)  # clamp for x == 1.0
        buckets[idx].append(x)
    result: List[float] = []
    for bucket in buckets:
        insertion_sort_float(bucket)
        result.extend(bucket)
    return result


def insertion_sort_float(arr: List[float]) -> None:
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
```

## Math

### Comparison-Sort Lower Bound via Decision Trees

Any deterministic comparison-based sorting algorithm on n elements can be modeled as a decision tree. Each internal node encodes one comparison `a[i] : a[j]` with two branches (≤ or >). Each leaf encodes one permutation of the input (one possible sorted order). The algorithm must be able to reach every possible permutation, so the tree must have at least n! leaves. A binary tree of height h has at most 2^h leaves, giving:

```math
2^h \geq n!
\implies h \geq \log_2(n!)
```

By Stirling's approximation:

```math
\log_2(n!) = \sum_{k=1}^{n} \log_2 k \approx n \log_2 n - n \log_2 e = \Theta(n \log n)
```

Therefore any comparison sort requires at least Ω(n log n) comparisons in the worst case. Merge sort and heapsort achieve this bound exactly.

A partial decision tree for sorting 3 elements {a, b, c}:

```
                  a:b
                 /   \
              a<b     a>b
              /           \
            b:c           b:c
           /   \         /   \
         b<c   b>c     b<c   b>c
          |     |       |     |
        [a,b,c] a:c  [b,a,c]  ...
               /   \
             a<c   a>c
              |     |
           [a,c,b] [c,a,b]

Leaves = 6 = 3!   Height ≥ ⌈log₂ 6⌉ = 3
```

### Master Theorem Applied to Merge Sort

The recurrence for merge sort is:

```math
T(n) = 2\,T\!\left(\frac{n}{2}\right) + O(n)
```

Applying the Master Theorem with a=2, b=2, f(n)=n:

```math
n^{\log_b a} = n^{\log_2 2} = n^1 = n = \Theta(f(n))
\implies T(n) = \Theta(n \log n)
```

### Average-Case Quicksort Recurrence

If the pivot is chosen uniformly at random from the n elements, the expected partition sizes are (k-1) and (n-k) for k uniform on [1,n]. The average-case recurrence is:

```math
T(n) = \frac{1}{n} \sum_{k=1}^{n} \left[T(k-1) + T(n-k)\right] + O(n) = \frac{2}{n}\sum_{k=0}^{n-1} T(k) + O(n)
```

Solving by substitution (or annihilator method) gives T(n) = O(n log n). Intuitively, a random pivot lands in the middle 50% of values with probability 1/2, giving a split no worse than (n/4, 3n/4) half the time — still O(n log n).

## Real-world Example

### Timsort in CPython — `list.sort()` and `sorted()`

Python's `list.sort()` and `sorted()` use Timsort, a hybrid algorithm designed by Tim Peters in 2002 specifically for real-world data that is rarely uniformly random. The algorithm detects existing sorted runs (ascending or descending subsequences), reverses descending runs in place, extends any run shorter than `minrun` (32–64 elements chosen to make the number of runs a power of 2) using binary insertion sort, then merges runs using a left-to-right merge queue with a "galloping" optimization: if one side wins 7 consecutive comparisons during a merge, switch to a binary search to skip large blocks at once.

```python
import time
from typing import Any


def timsort_behavior_demo() -> None:
    """Show Timsort exploiting pre-existing structure."""
    import random

    n = 1_000_000

    # Case 1: already sorted — Timsort is O(n), one run detected
    data_sorted: list[int] = list(range(n))
    t0 = time.perf_counter()
    data_sorted.sort()
    t_sorted = time.perf_counter() - t0

    # Case 2: nearly sorted — a few random swaps
    data_nearly: list[int] = list(range(n))
    for _ in range(1000):
        i, j = random.randrange(n), random.randrange(n)
        data_nearly[i], data_nearly[j] = data_nearly[j], data_nearly[i]
    t0 = time.perf_counter()
    data_nearly.sort()
    t_nearly = time.perf_counter() - t0

    # Case 3: random — no runs to exploit
    data_random: list[int] = random.sample(range(n), n)
    t0 = time.perf_counter()
    data_random.sort()
    t_random = time.perf_counter() - t0

    print(f"Sorted input:  {t_sorted*1000:.1f} ms")
    print(f"Nearly sorted: {t_nearly*1000:.1f} ms")
    print(f"Random:        {t_random*1000:.1f} ms")
    # Expected: sorted ≈ 10–30 ms, nearly sorted ≈ 30–60 ms, random ≈ 150–300 ms


timsort_behavior_demo()
```

> [!TIP]
> Use `key=` on `list.sort()` and `sorted()` instead of a `cmp` function or manual decoration. Python evaluates the `key` once per element (O(n) calls), then sorts the (key, element) pairs using the fast C implementation. The decorate-sort-undecorate (DSU) idiom is built in: `sorted(records, key=lambda r: r.score)` avoids writing a comparator and lets Timsort operate on plain integers or tuples.

### Sort-Merge Join in Databases

The sort-merge join is one of three physical join algorithms in relational databases (alongside hash join and nested-loop join). Both input relations are sorted on the join key, then a linear two-pointer merge identifies matching tuples. Sort cost is O((n + m) log(n + m)) with external merge sort if the data does not fit in memory; the merge itself is O(n + m). Sort-merge join outperforms hash join when the inputs are already sorted (from an index scan) or when the join produces a large fraction of the cross product (low selectivity).

```python
from typing import Iterator


def sort_merge_join(
    left: list[dict[str, Any]],
    right: list[dict[str, Any]],
    key: str,
) -> Iterator[tuple[dict[str, Any], dict[str, Any]]]:
    """
    Sort-merge equi-join on `key`. Both sides sorted ascending by key.
    O((n + m) log(n + m)) sort + O(n + m) merge.
    """
    left_sorted = sorted(left, key=lambda r: r[key])
    right_sorted = sorted(right, key=lambda r: r[key])
    i = j = 0
    while i < len(left_sorted) and j < len(right_sorted):
        lk = left_sorted[i][key]
        rk = right_sorted[j][key]
        if lk < rk:
            i += 1
        elif lk > rk:
            j += 1
        else:
            # collect all right-side duplicates for current key
            j_start = j
            while j < len(right_sorted) and right_sorted[j][key] == lk:
                yield left_sorted[i], right_sorted[j]
                j += 1
            # handle left-side duplicates: reset right pointer
            i += 1
            if i < len(left_sorted) and left_sorted[i][key] == lk:
                j = j_start


# Example
orders = [{"order_id": 1, "cid": 10}, {"order_id": 2, "cid": 20}]
customers = [{"cid": 10, "name": "Alice"}, {"cid": 20, "name": "Bob"}]
for o, c in sort_merge_join(orders, customers, "cid"):
    print(o["order_id"], c["name"])
```

## In Practice

Python's `list.sort()` is implemented in C in CPython (Objects/listobject.c, the `listsort.txt` document by Tim Peters is the canonical reference). It is always Timsort. The `sorted()` built-in calls the same algorithm but returns a new list. Both accept a `key=` callable and a `reverse=` flag; neither accepts a `cmp=` function (removed in Python 3 — use `functools.cmp_to_key` if you must port Python 2 code).

C's `qsort` is typically an introsort (quicksort + heapsort fallback + insertion sort for small subarrays) and is **not stable** — do not rely on it for stable ordering of records with equal keys. On Linux, glibc's `qsort` is actually a merge sort variant (stable) but this is implementation-specific.

When to use counting/radix sort: if your keys are bounded integers and k ≤ O(n log n / log log n), the linear-time sorts pay off. In practice: sorting 32-bit integers with radix-256 takes 4 passes × O(n + 256) = O(n), versus O(n log n) for comparison sort. For n > ~100k elements on 32-bit keys, radix sort wins.

> [!TIP]
> For k-sorted arrays (each element is at most k positions from its sorted position), insertion sort runs in O(nk) — far better than its O(n²) worst case. Use a min-heap of size k+1 for an O(n log k) solution: push elements one by one, always popping the minimum. This is the standard "sort a nearly-sorted stream" idiom used in merge-k-sorted-lists and priority-queue-based external sort.

> [!WARNING]
> Quicksort on already-sorted (or reverse-sorted) input with the last or first element as pivot produces O(n²) behaviour — every partition is maximally unbalanced. Mitigations: median-of-three pivot selection; random pivot; introsort (switch to heapsort after recursion depth exceeds 2 log n). Python's `list.sort()` does not use quicksort, so this is a C/Java/Go concern.

> [!CAUTION]
> Recursion depth for naive recursive quicksort on n=10^6 elements is O(n) worst case, which blows the default stack on most platforms (Python's default recursion limit is 1000; C's typical stack is 1–8 MB). Always implement quicksort with explicit tail-call optimization or switch to an iterative version with an explicit stack for the larger partition.

## Pitfalls

- **"Stable sort preserves all element order."** — It preserves the relative order of *equal* elements only. Elements with distinct keys can be reordered arbitrarily.

- **"Selection sort is stable because it finds the minimum without swapping."** — The swap of the minimum into the front position can move an element past duplicates of itself. Example: [3a, 3b, 1] → selection sort swaps 1 into position 0, giving [1, 3b, 3a] — 3a and 3b are reversed.

- **"Merge sort's stability requires no special care."** — Stability requires `if left[i] <= right[j]` (prefer left on ties), not `if left[i] < right[j]`. Using strict `<` makes the merge prefer the right-half element on ties, reversing the relative order and breaking stability.

- **"Quicksort is always O(n log n)."** — Only on average with a good pivot. On sorted or reverse-sorted input with a fixed pivot strategy, it degrades to O(n²). Production quicksorts use random pivot or median-of-three.

- **"Heapsort is the best in-place O(n log n) sort."** — Heapsort's access pattern is cache-hostile: extracting the max and heapifying causes far-apart memory accesses. In practice, on modern CPUs with deep cache hierarchies, merge sort with a buffer often beats heapsort despite using O(n) extra memory, because sequential access patterns allow hardware prefetching.

- **"Counting sort works for any integer keys."** — Only for non-negative keys in a known bounded range. If the range k is much larger than n, the count array wastes memory and dominates runtime. For 64-bit keys, radix sort is the right tool, not counting sort.

- **"Radix sort is always faster than comparison sort."** — For small n or large word width w, the constant factors make radix sort slower. Radix sort shines at n ≥ ~10^5 for 32-bit keys with base 256 (4 passes). For n < 1000 or keys with many bits, comparison sort typically wins.

- **"Timsort is a merge sort."** — Timsort is a *hybrid*: natural mergesort for run detection and merging, binary insertion sort for extending short runs. Its galloping merge is an optimisation not present in textbook merge sort. The run-detection phase makes it adaptive in a way that pure merge sort is not.

## Exercises

### Exercise 1 — Sort 0s, 1s, and 2s (Dutch National Flag)

Given an array containing only 0s, 1s, and 2s, sort it in O(n) time and O(1) space without using counting sort (which would require two passes).

> [!TIP]
> Maintain three pointers: `lo` (boundary of 0s), `mid` (current element), `hi` (boundary of 2s). Invariant: arr[0..lo-1] = 0, arr[lo..mid-1] = 1, arr[hi+1..n-1] = 2.

```python
from typing import List


def sort_colors(arr: List[int]) -> None:
    """Dutch National Flag. O(n) time, O(1) space. One pass."""
    lo, mid, hi = 0, 0, len(arr) - 1
    while mid <= hi:
        if arr[mid] == 0:
            arr[lo], arr[mid] = arr[mid], arr[lo]
            lo += 1
            mid += 1
        elif arr[mid] == 1:
            mid += 1
        else:  # arr[mid] == 2
            arr[mid], arr[hi] = arr[hi], arr[mid]
            hi -= 1
            # do NOT increment mid: newly swapped element needs checking


# Test
a = [2, 0, 2, 1, 1, 0]
sort_colors(a)
assert a == [0, 0, 1, 1, 2, 2]
```

Complexity: O(n) time, O(1) space. One pass through the array.

### Exercise 2 — Kth Largest Element

Find the kth largest element in an unsorted array without fully sorting it.

> [!TIP]
> Quickselect (a partition-based selection algorithm) gives O(n) average time. Alternatively, maintain a min-heap of size k: the heap root is always the kth largest seen so far. The heap approach gives O(n log k) and works in a streaming setting.

```python
import random
from typing import List


def kth_largest(arr: List[int], k: int) -> int:
    """Quickselect. O(n) average, O(n²) worst. In-place (modifies arr copy)."""
    nums = list(arr)
    target = len(nums) - k  # kth largest = (n-k)th smallest (0-indexed)

    def quickselect(l: int, r: int) -> int:
        pivot_idx = random.randint(l, r)
        nums[pivot_idx], nums[r] = nums[r], nums[pivot_idx]
        pivot = nums[r]
        store = l
        for i in range(l, r):
            if nums[i] <= pivot:
                nums[store], nums[i] = nums[i], nums[store]
                store += 1
        nums[store], nums[r] = nums[r], nums[store]
        if store == target:
            return nums[store]
        elif store < target:
            return quickselect(store + 1, r)
        else:
            return quickselect(l, store - 1)

    return quickselect(0, len(nums) - 1)


assert kth_largest([3, 2, 1, 5, 6, 4], 2) == 5
assert kth_largest([3, 2, 3, 1, 2, 4, 5, 5, 6], 4) == 4
```

Complexity: O(n) average (randomized pivot), O(n²) worst, O(1) extra space.

### Exercise 3 — Merge K Sorted Lists

Given k sorted arrays, merge them into a single sorted array.

> [!TIP]
> Use a min-heap of size k. Push the first element of each list. On each extraction, push the next element from the same list. This gives O(n log k) where n is the total number of elements.

```python
import heapq
from typing import List


def merge_k_sorted(lists: List[List[int]]) -> List[int]:
    """Merge k sorted lists. O(n log k) time, O(k) heap space."""
    result: List[int] = []
    # heap entries: (value, list_index, element_index_within_list)
    heap: List[tuple[int, int, int]] = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))
    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)
        next_idx = elem_idx + 1
        if next_idx < len(lists[list_idx]):
            heapq.heappush(heap, (lists[list_idx][next_idx], list_idx, next_idx))
    return result


assert merge_k_sorted([[1, 4, 7], [2, 5, 8], [3, 6, 9]]) == list(range(1, 10))
```

Complexity: O(n log k) time, O(k) space.

### Exercise 4 — Count Inversions

Count the number of inversions in an array (pairs i < j with arr[i] > arr[j]).

> [!TIP]
> Modify the merge step of merge sort: whenever a right-half element is copied before a left-half element, all remaining left-half elements form inversions with it. Add `n1 - i` to the count (where i is the current left pointer).

```python
from typing import List, Tuple


def count_inversions(arr: List[int]) -> int:
    """Count inversions via modified merge sort. O(n log n) time, O(n) space."""
    _, count = _merge_count(list(arr))
    return count


def _merge_count(arr: List[int]) -> Tuple[List[int], int]:
    if len(arr) <= 1:
        return arr, 0
    mid = len(arr) // 2
    left, lc = _merge_count(arr[:mid])
    right, rc = _merge_count(arr[mid:])
    merged, mc = _merge_and_count(left, right)
    return merged, lc + rc + mc


def _merge_and_count(left: List[int], right: List[int]) -> Tuple[List[int], int]:
    result: List[int] = []
    inversions = 0
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            # left[i] > right[j]: all remaining left elements form inversions
            result.append(right[j])
            inversions += len(left) - i
            j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result, inversions


assert count_inversions([2, 4, 1, 3, 5]) == 3   # (4,1),(4,3),(2,1)
assert count_inversions([10, 20, 30, 40]) == 0
assert count_inversions([40, 30, 20, 10]) == 6   # n(n-1)/2 = 4*3/2
```

Complexity: O(n log n) time, O(n) space (same as merge sort).

### Exercise 5 — Sort a Nearly-Sorted (K-Sorted) Array

Given an array where each element is at most k positions away from its sorted position, sort it efficiently.

> [!TIP]
> Use a min-heap of size k+1. Slide a window of k+1 elements across the array: always extract the minimum from the heap (it must be the smallest remaining unplaced element) and push the next unseen element.

```python
import heapq
from typing import List


def sort_k_sorted(arr: List[int], k: int) -> List[int]:
    """Sort k-sorted array. O(n log k) time, O(k) space."""
    heap: List[int] = arr[: k + 1]
    heapq.heapify(heap)  # O(k)
    result: List[int] = []
    for i in range(k + 1, len(arr)):
        result.append(heapq.heappushpop(heap, arr[i]))
    while heap:
        result.append(heapq.heappop(heap))
    return result


# Each element at most 3 positions from its sorted position
assert sort_k_sorted([6, 5, 3, 2, 8, 10, 9], 3) == [2, 3, 5, 6, 8, 9, 10]
```

Complexity: O(n log k) time, O(k) space. Beats O(n log n) when k << n.

### Exercise 6 — External Sort (Conceptual)

You have a 1 TB file of 64-bit integers on disk. Your RAM holds 1 GB. How do you sort the file?

**Approach — External Merge Sort:**

Phase 1 (run generation): Read 1 GB chunks into RAM, sort each with an in-memory sort (Timsort/quicksort), write sorted runs back to disk. A 1 TB file produces ~1000 sorted runs of 1 GB each.

Phase 2 (k-way merge): Open all 1000 run files simultaneously. Use a min-heap of 1000 entries (one per run). Repeatedly extract the global minimum and write it to the output file, pushing the next element from the same run into the heap.

```
Memory layout during k-way merge:
┌─────────────────────────────────────────────────────┐
│  RAM (1 GB)                                         │
│  ┌──────────┐ ┌──────────┐     ┌──────────┐        │
│  │ Run 1    │ │ Run 2    │ ... │ Run 1000 │  ← input│
│  │ buffer   │ │ buffer   │     │ buffer   │  buffers│
│  │ (1 MB)   │ │ (1 MB)   │     │ (1 MB)   │        │
│  └──────────┘ └──────────┘     └──────────┘        │
│  ┌──────────────────────────────────────────┐       │
│  │           Output buffer (1 MB)           │ ← out │
│  └──────────────────────────────────────────┘       │
│  ┌─────────────────┐                                │
│  │  Min-heap       │ 1000 entries × 8 bytes ≈ 8 KB │
│  └─────────────────┘                                │
└─────────────────────────────────────────────────────┘
```

Complexity: O((N/M) log(N/M) × M log M) ≈ O(N log N) total comparisons; 2 × N/B disk I/Os per pass where B is block size. With a 1 MB disk block and 1 TB data: ~2 million I/Os for phase 2. A second merge pass can be avoided if N/M ≤ M/B (the number of runs fits in one merge step).

> [!NOTE]
> Replacement selection (using a priority queue during run generation) can produce average run lengths of 2M instead of M, halving the number of phase-2 runs. This is the technique used by classic database systems like PostgreSQL and Oracle for external sort operations.

## Sources

- Knuth, D.E. *The Art of Computer Programming, Vol. 3: Sorting and Searching* (2nd ed., 1998). The canonical reference for every classical sort algorithm.
- Peters, T. *listsort.txt* — the definitive description of Timsort: https://github.com/python/cpython/blob/main/Objects/listsort.txt
- Cormen, T.H. et al. *Introduction to Algorithms* (CLRS), 4th ed. Chapters 6 (Heapsort), 7 (Quicksort), 8 (Linear-time sorting). MIT Press, 2022.
- Hoare, C.A.R. "Quicksort." *Computer Journal* 5(1):10–15, 1962. Original paper.
- Blum, M. et al. "Time Bounds for Selection." *Journal of Computer and System Sciences* 7(4):448–461, 1973. Median-of-medians algorithm (O(n) worst-case selection).
- CPython source: Objects/listobject.c — https://github.com/python/cpython/blob/main/Objects/listobject.c
- GFG Lecture notes — Sorting (PDF), course from which these notes are derived. Conversation with user on 2026-05-19.

## Related

- [[02-arrays-and-searching]]
- [[04-hashing]]
- [[09-trees]]
