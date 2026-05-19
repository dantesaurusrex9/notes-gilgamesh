---
title: "Linked Lists"
created: 2026-05-19
updated: 2026-05-19
tags: [data-structures, linked-list, pointers, floyd, two-pointer, algorithms]
aliases: [linked-list, singly-linked-list, doubly-linked-list]
---

# Linked Lists

[toc]

> **TL;DR:** A linked list is a sequence of heap-allocated nodes, each carrying data and one or two pointers to neighbours, giving O(1) insert/delete at any known node at the cost of O(n) random access and poor cache locality. Arrays own contiguous memory and win on read-heavy workloads; linked lists win when insertions and deletions dominate and indices do not matter.

## Vocabulary

**Node** — the atomic unit: a heap allocation holding a payload (`data`) and one or two pointer fields (`next`, optionally `prev`).

**Head** — the pointer to the first node. Losing this reference means losing the whole list.

**Tail** — the pointer to the last node. Maintaining it explicitly makes append O(1); without it, append is O(n).

**Sentinel / dummy node** — a permanently-allocated node with no meaningful data, placed before the head (or after the tail) to eliminate edge cases for empty-list checks and boundary insertions.

**Singly linked list** — each node holds one forward pointer (`next`). Traversal is one-directional; deletion requires knowledge of the predecessor.

**Doubly linked list** — each node holds both `next` and `prev`. O(1) delete given only the node pointer; doubles per-node pointer overhead.

**Circular linked list** — the tail's `next` wraps back to the head (singly) or the head's `prev` wraps to the tail (doubly). No null terminator; requires a sentinel or loop-termination condition.

**Slow/fast pointer (Floyd's tortoise & hare)** — two pointers advancing at speeds 1 and 2 steps per iteration. Used for cycle detection, finding the middle node, and finding the k-th node from the end.

**In-place reversal** — reversing arrow directions by rewiring `next` (and `prev`) pointers without allocating a new list. O(n) time, O(1) space.

**Cycle / loop** — a list where some node's `next` points to an earlier node, forming an infinite traversal path.

**Cycle entry node** — the first node inside a cycle (the node where the loop begins), distinct from the meeting point of slow and fast pointers.

---

## Intuition

Arrays and linked lists solve the same abstract problem — ordered sequence — but make opposite bets on hardware. An array bets that you will read and index often; it packs all elements contiguously so the CPU prefetcher can pull a cache line of 64 bytes and serve 16 consecutive `int32` reads from L1. A linked list bets that you will insert and delete often at known positions; it never has to shift anything, because each node is independently allocated and re-linked in O(1).

The trouble is that modern hardware punishes pointer-chasing mercilessly. Each node dereference is a potential L1/L2 miss (5–300 ns) because nodes land at arbitrary heap addresses. Arrays almost always win in practice — even operations nominally O(n) on arrays (like insert-at-head with a shift) outrun a linked list traversal when n is small due to cache effects. The canonical use cases where linked lists genuinely win are: an LRU cache eviction policy (O(1) move-to-front on a doubly linked list + hash map), a deque where both ends see heavy traffic, and intrusive list patterns in OS kernels where nodes embed the link fields directly in their data structs (zero allocation overhead).

> [!NOTE]
> Python's built-in `list` is a **dynamic array** (realloc + memmove), not a linked list. `collections.deque` is a doubly-linked list of fixed-size blocks (not individual nodes). There is no standard-library singly linked list in Python — you implement one yourself.

---

## Memory layout

**Figure 1:** Contiguous array vs. scattered linked list nodes in memory.

```
ARRAY — contiguous, cache-friendly
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
 0x100 0x104 0x108 0x10C 0x110
  ↑ All 5 elements fit in one 64-byte cache line

LINKED LIST — scattered, pointer-chased
  0x1A0          0x3F8          0x720
┌──────┬──────┐  ┌──────┬──────┐  ┌──────┬──────┐
│  10  │0x3F8 │→ │  20  │0x720 │→ │  30  │ None │
└──────┴──────┘  └──────┴──────┘  └──────┴──────┘
  data   next       data   next       data   next
  (each node likely in a different cache line)
```

**Figure 2:** Node box notation used throughout these notes.

```
Singly linked node:        Doubly linked node:
┌──────┬──────┐     ┌──────┬──────┬──────┐
│ data │ next │     │ prev │ data │ next │
└──────┴──────┘     └──────┴──────┴──────┘
```

**Figure 3:** Reversing a singly linked list — arrows flip step by step.

```
BEFORE:
head
 ↓
[10│→] → [20│→] → [30│→] → [40│None]

STEP 1: prev=None, curr=10, save next=20, point 10→None
         [10│None]  [20│→] → [30│→] → [40│None]
          ↑prev=curr  curr=next

STEP 2: prev=10, curr=20, save next=30, point 20→10
         [10│None] ← [20│→10]  [30│→] → [40│None]

STEP 3: prev=20, curr=30, point 30→20
STEP 4: prev=30, curr=40, point 40→30, curr becomes None → stop

AFTER:
         head
          ↓
[40│→30] → [30│→20] → [20│→10] → [10│None]
```

**Figure 4:** Floyd's slow/fast pointer — frame-by-frame on a cyclic list.

```
List: 10 → 12 → 15 → 20 → 35 → 25 → (back to 15)
       s                              cycle entry = 15

Step 0:  slow=10   fast=10
Step 1:  slow=12   fast=15
Step 2:  slow=15   fast=25
Step 3:  slow=20   fast=15   ← fast re-entered cycle
Step 4:  slow=35   fast=25
Step 5:  slow=25   fast=25   ← MEET (inside cycle)
```

---

## Math foundations

The five subsections below give you the formal tools to *reason about* linked-list algorithms — not just run them. Loop invariants let you prove correctness; the Floyd derivation shows exactly why the tortoise-and-hare trick works; the random-walk framing quantifies the O(n) access cost; the amortized counting argument justifies why a doubly linked list achieves O(1) per op; and the cache-penalty numbers turn the abstract "poor locality" warning into concrete wall-clock damage.

### Loop invariants

A loop invariant is a predicate about program state that is true (a) before the loop begins, (b) preserved across every iteration, and (c) still true at loop exit — where the invariant plus the termination condition together imply correctness. This three-part structure — initialization, maintenance, termination — is the standard formal method for verifying any iterative linked-list algorithm, from traversal to in-place reversal.

**Worked example: in-place reversal invariant.**

Define I(k): "after iteration k, `prev` points to the head of the reversed sublist consisting of the first k original nodes, and `curr` points to the (k+1)-th original node (or None if exhausted)."

- **Initialization (k = 0):** Before the loop, `prev = None` and `curr = head`. The reversed sublist of the first 0 nodes is empty, so its head is correctly None. `curr` is the 1st original node. I(0) holds.
- **Maintenance:** Assume I(k) holds. The body saves `next_node = curr.next`, sets `curr.next = prev`, then advances `prev = curr` and `curr = next_node`. After this, `prev` is the head of the reversed sublist of the first k+1 nodes, and `curr` is the (k+2)-th original node. I(k+1) holds.
- **Termination:** The loop exits when `curr is None` (k = n). By I(n), `prev` is the head of the fully reversed list.

```python
def reverse_with_invariant_annotation(head):
    # I(0): prev=None, curr=head — reversed sublist of 0 nodes is empty
    prev = None
    curr = head
    while curr is not None:
        # I(k) holds here
        next_node = curr.next   # save before overwrite
        curr.next = prev        # reverse the arrow
        prev = curr             # prev becomes new reversed-head
        curr = next_node        # curr advances to next original node
        # I(k+1) holds here
    # curr is None => k=n; prev is head of fully reversed list
    return prev
```

### Floyd's cycle-detection math

The tortoise-and-hare algorithm is often presented as a trick that "just works." The derivation is short and worth knowing: it tells you both *why* the pointers meet and *why* resetting slow to the head finds the cycle entry, with no handwaving.

Let the list have a non-cyclic prefix of length μ nodes and a cycle of length C. After t steps, slow is at position t and fast is at position 2t. For t ≥ μ, both are inside the cycle. Slow's cycle-offset is (t − μ) mod C; fast's is (2t − μ) mod C. Setting them equal:

```math
(2t - \mu) \equiv (t - \mu) \pmod{C}
```

```math
t \equiv 0 \pmod{C}
```

So the first meeting occurs at the smallest t ≥ μ that is a multiple of C — i.e., t = C · ⌈μ/C⌉. The meeting is guaranteed in at most μ + C steps.

**Why phase 2 finds the cycle entry.** At the meeting point, slow has traveled t steps (t ≡ 0 mod C, t ≥ μ). The meeting point's cycle-offset is (t − μ) mod C. Because t ≡ 0 mod C:

```math
(t - \mu) \bmod C \;=\; (-\mu) \bmod C \;=\; C - (\mu \bmod C)
```

The meeting point is C − (μ mod C) steps ahead of the cycle entry, equivalently μ mod C steps *before* it. Reset slow to the head. Advance both at speed 1. After μ steps, slow is at the cycle entry. Fast's new cycle-offset is:

```math
\bigl(C - (\mu \bmod C)\bigr) + \mu \;=\; C + \mu - (\mu \bmod C) \;\equiv\; 0 \pmod{C}
```

Cycle-offset 0 is the cycle entry. Both pointers arrive simultaneously.

> [!IMPORTANT]
> Phase 2 requires both pointers to advance at speed 1 after the reset. Leaving fast at speed 2 breaks the derivation — the pointers will not meet at the cycle entry.

### Expected work in random linked-list traversal

A uniform random lookup in an unsorted linked list of n nodes requires scanning from the head with no index or shortcut. If the target is equally likely to occupy any position, the expected number of hops is the mean of a uniform draw from {1, 2, …, n}:

```math
\mathbb{E}[\text{hops to target}] = \frac{1}{n} \sum_{i=1}^{n} i = \frac{n+1}{2} = \Theta(n)
```

Compare to a balanced BST, where each comparison halves the remaining search space:

```math
\mathbb{E}[\text{comparisons in BST}] = \Theta(\log_{2} n)
```

For n = 1,000,000 the list needs ~500,000 hops on average; the BST needs ~20. This gap is the precise, quantified reason linked-list lookup is called O(n): there is no index, no binary-search lever, no spatial locality to exploit. The structure trades O(n) access for O(1) insert/delete at a known node — a deliberate asymmetry, not a design flaw.

> [!NOTE]
> Self-organizing lists (move-to-front heuristic) reduce the *expected* hops for skewed access distributions. If the top 20% of keys account for 80% of lookups, move-to-front pushes the expected hop count well below n/2. This is the informal basis of LRU-approximation caches.

### Why doubly linked lists enable O(1) amortized operations

The core insight is a counting argument: over any sequence of n insertions and m deletions on a doubly linked list, each node is touched by exactly one insertion and at most one deletion. Both operations cost a bounded constant number of pointer writes (at most 6 per op). The total pointer operations across the entire sequence is therefore bounded:

```math
\text{total pointer ops} \;\leq\; (n + m) \cdot c \quad (c \leq 6)
```

Dividing over n + m operations gives O(1) amortized cost per operation. This counting argument is why LRU caches, deques, and OS scheduler run-queues built on doubly linked lists achieve O(1) per-operation throughput: each node's pointer fields are modified only at its birth and death, no matter how many other operations happen in between.

```python
def count_pointer_ops(n_inserts: int, n_deletes: int) -> int:
    """
    Conservative upper bound on pointer writes for n inserts + n deletes
    on a doubly linked list (sentinel-based, so no boundary branches).

    insert_at_head: node.next, node.prev, old_head.prev, self.head = 4 writes
    delete_node:    prev.next, next.prev, node.prev=None, node.next=None = 4 writes
    """
    ops_per_insert = 4
    ops_per_delete = 4
    return n_inserts * ops_per_insert + n_deletes * ops_per_delete

# 1M inserts + 1M deletes = 8M pointer writes total — O(n), not O(n^2)
print(count_pointer_ops(1_000_000, 1_000_000))  # 8000000
```

### Cache locality: the pointer-chasing penalty

Walking a linked list incurs one potential L2/L3 cache miss per node because consecutive nodes occupy arbitrary, unrelated heap addresses — the allocator cannot guarantee contiguity. A contiguous array, by contrast, benefits from spatial prefetching: a single 64-byte cache line holds 8 × `int64` values (or 16 × `int32`), so the effective miss rate is 8–16x lower for sequential access.

Concrete wall-clock comparison for a 1M-element structure traversed sequentially with random allocation:

| Structure | Cache misses (approx.) | Cost per miss | Total time (approx.) |
| :--- | ---: | ---: | ---: |
| Linked list, 1M nodes | ~1,000,000 | 50–300 ns | **50–300 ms** |
| Array, 1M × int64 | ~125,000 cache lines | ~1 ns | **< 1 ms** |

> [!IMPORTANT]
> Cache miss penalty dominates linked-list traversal in practice. A 1M-node linked list scan is 50–300x slower than an equivalent array scan on modern hardware, even though both are O(n) algorithmically. The constants differ by two orders of magnitude. This is why the comparison table shows a "tie" at O(n) for traversal yet arrays win decisively in every benchmark — asymptotic analysis hides the hardware cost.

---

## How it works

Linked list operations all reduce to pointer rewiring. The key discipline: always save the `next` pointer before overwriting it, and always verify `None` before dereferencing. Every operation below is O(1) amortized if you have a direct handle to the relevant nodes; O(n) if you must walk to find them.

### Singly linked list

A singly linked list has a `head` pointer and optionally a `tail` pointer. The Python `Node` class below uses `__slots__` to skip the per-instance `__dict__`, cutting memory from ~240 bytes per node down to ~56 bytes (two object references + object header on CPython 3.12).

```python
from __future__ import annotations
from typing import Optional, Any


class Node:
    """Single node for a singly linked list."""
    __slots__ = ("data", "next")

    def __init__(self, data: Any) -> None:
        self.data: Any = data
        self.next: Optional[Node] = None


class LinkedList:
    """Singly linked list with O(1) head and tail access."""
    __slots__ = ("head", "tail", "length")

    def __init__(self) -> None:
        self.head: Optional[Node] = None
        self.tail: Optional[Node] = None
        self.length: int = 0

    # ── insertion ─────────────────────────────────────────────────────────

    def insert_at_head(self, data: Any) -> None:
        """O(1). New node becomes the new head."""
        node = Node(data)
        node.next = self.head
        self.head = node
        if self.tail is None:          # first insertion
            self.tail = node
        self.length += 1

    def insert_at_tail(self, data: Any) -> None:
        """O(1) with tail pointer, O(n) without."""
        node = Node(data)
        if self.tail is None:
            self.head = self.tail = node
        else:
            self.tail.next = node
            self.tail = node
        self.length += 1

    def insert_at_position(self, data: Any, pos: int) -> None:
        """O(pos). Insert before the node currently at index pos (0-based)."""
        if pos == 0:
            self.insert_at_head(data)
            return
        node = Node(data)
        curr: Optional[Node] = self.head
        for _ in range(pos - 2):      # walk to pos-1
            if curr is None:
                raise IndexError("position out of range")
            curr = curr.next
        if curr is None:
            raise IndexError("position out of range")
        node.next = curr.next
        curr.next = node
        if node.next is None:          # inserted at tail
            self.tail = node
        self.length += 1

    # ── deletion ──────────────────────────────────────────────────────────

    def delete_head(self) -> Any:
        """O(1)."""
        if self.head is None:
            raise ValueError("delete from empty list")
        data = self.head.data
        self.head = self.head.next
        if self.head is None:
            self.tail = None
        self.length -= 1
        return data

    def delete_tail(self) -> Any:
        """O(n) for singly linked — must walk to find predecessor."""
        if self.head is None:
            raise ValueError("delete from empty list")
        if self.head is self.tail:    # single-node list
            data = self.head.data
            self.head = self.tail = None
            self.length -= 1
            return data
        curr: Optional[Node] = self.head
        while curr is not None and curr.next is not self.tail:
            curr = curr.next
        assert curr is not None
        data = self.tail.data          # type: ignore[union-attr]
        curr.next = None
        self.tail = curr
        self.length -= 1
        return data

    def delete_value(self, value: Any) -> bool:
        """O(n). Returns True if the value was found and removed."""
        if self.head is None:
            return False
        if self.head.data == value:
            self.delete_head()
            return True
        curr: Optional[Node] = self.head
        while curr is not None and curr.next is not None:
            if curr.next.data == value:
                if curr.next is self.tail:
                    self.tail = curr
                curr.next = curr.next.next
                self.length -= 1
                return True
            curr = curr.next
        return False

    # ── traversal ─────────────────────────────────────────────────────────

    def to_list(self) -> list[Any]:
        result: list[Any] = []
        curr = self.head
        while curr is not None:
            result.append(curr.data)
            curr = curr.next
        return result

    def __repr__(self) -> str:
        return " → ".join(str(x) for x in self.to_list()) + " → None"
```

### Doubly linked list

The doubly linked node adds a `prev` pointer. This makes delete-given-node O(1) (no predecessor walk needed) and enables bidirectional traversal. The cost is one extra pointer per node (8 bytes on 64-bit) and more pointer surgery per operation — every link change must update both `next` and `prev`.

```python
class DNode:
    """Node for a doubly linked list."""
    __slots__ = ("data", "prev", "next")

    def __init__(self, data: Any) -> None:
        self.data: Any = data
        self.prev: Optional[DNode] = None
        self.next: Optional[DNode] = None


class DoublyLinkedList:
    """Doubly linked list — O(1) insert/delete at head or tail."""
    __slots__ = ("head", "tail", "length")

    def __init__(self) -> None:
        self.head: Optional[DNode] = None
        self.tail: Optional[DNode] = None
        self.length: int = 0

    def insert_at_head(self, data: Any) -> DNode:
        node = DNode(data)
        if self.head is None:
            self.head = self.tail = node
        else:
            node.next = self.head
            self.head.prev = node
            self.head = node
        self.length += 1
        return node

    def insert_at_tail(self, data: Any) -> DNode:
        node = DNode(data)
        if self.tail is None:
            self.head = self.tail = node
        else:
            node.prev = self.tail
            self.tail.next = node
            self.tail = node
        self.length += 1
        return node

    def delete_node(self, node: DNode) -> None:
        """O(1) delete given a direct node reference — the DLL superpower."""
        if node.prev is not None:
            node.prev.next = node.next
        else:
            self.head = node.next        # node was head
        if node.next is not None:
            node.next.prev = node.prev
        else:
            self.tail = node.prev        # node was tail
        node.prev = node.next = None
        self.length -= 1

    def reverse(self) -> None:
        """O(n) in-place reversal: swap prev/next on every node."""
        curr = self.head
        while curr is not None:
            curr.prev, curr.next = curr.next, curr.prev
            curr = curr.prev             # move via old next (now prev)
        self.head, self.tail = self.tail, self.head
```

> [!IMPORTANT]
> In a doubly linked list reversal, after swapping `prev` and `next` on each node, advance with `curr = curr.prev` — because `prev` now holds what used to be `next`. Using `curr = curr.next` would walk backward through the unreversed portion.

### Circular linked list

The tail's `next` points back to the head instead of `None`. This removes the sentinel-`None` terminator and enables "wrap-around" traversal from any node. Traversal must stop when `curr.next == head` (not when `curr.next is None`). The canonical advantage: you can traverse the whole list starting from any node, which is useful for round-robin schedulers and ring buffers.

```python
class CircularLinkedList:
    """Singly circular linked list. tail.next always == head."""
    __slots__ = ("head", "tail", "length")

    def __init__(self) -> None:
        self.head: Optional[Node] = None
        self.tail: Optional[Node] = None
        self.length: int = 0

    def insert_at_tail(self, data: Any) -> None:
        node = Node(data)
        if self.head is None:
            self.head = self.tail = node
            node.next = node             # point to itself
        else:
            node.next = self.head        # wrap around
            self.tail.next = node
            self.tail = node
        self.length += 1

    def insert_at_head(self, data: Any) -> None:
        node = Node(data)
        if self.head is None:
            self.head = self.tail = node
            node.next = node
        else:
            node.next = self.head
            self.tail.next = node        # tail still wraps
            self.head = node
        self.length += 1

    def traverse(self) -> list[Any]:
        """Collect all values in one loop of the circle."""
        result: list[Any] = []
        if self.head is None:
            return result
        curr: Optional[Node] = self.head
        while True:
            assert curr is not None
            result.append(curr.data)
            curr = curr.next
            if curr is self.head:
                break
        return result
```

### Slow/fast pointer technique

The two-pointer trick exploits the fact that a fast pointer advancing at 2x the speed of a slow pointer will, in a cyclic structure, necessarily lap the slow pointer. The same relative-speed idea finds the middle (slow stops at n/2 when fast hits the end) and the k-th node from the end (advance fast by k first, then move both together until fast hits None).

```python
def find_middle(head: Optional[Node]) -> Optional[Node]:
    """O(n) time, O(1) space. Returns the middle node.
    For even-length lists returns the second of the two middle nodes."""
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
    return slow


def has_cycle(head: Optional[Node]) -> bool:
    """Floyd's cycle detection. O(n) time, O(1) space."""
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
        if slow is fast:
            return True
    return False


def find_cycle_start(head: Optional[Node]) -> Optional[Node]:
    """Returns the node where the cycle begins, or None.
    Phase 1: detect meeting point. Phase 2: reset slow to head,
    advance both one step at a time — they meet at the cycle entry."""
    slow = fast = head
    # phase 1: detect
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
        if slow is fast:
            break
    else:
        return None               # no cycle
    # phase 2: find entry
    slow = head
    while slow is not fast:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next          # type: ignore[union-attr]
    return slow
```

---

## Math

### Cycle detection meeting point

Let the linked list have a non-cyclic prefix of length `m` nodes and a cycle of length `k` nodes. The slow pointer enters the cycle at step `m`; by then the fast pointer has advanced `2m` steps and is `(2m - m) mod k = m mod k` steps into the cycle.

```math
\text{distance fast is ahead of slow (in cycle)} = m \bmod k
```

At each subsequent step, slow moves +1 and fast moves +2, so fast gains 1 position on slow per step (mod k). They meet when the gap closes to 0, i.e., after exactly `k - (m mod k)` more steps.

```math
\text{steps to meeting} = k - (m \bmod k)
```

**Why phase 2 works:** at the meeting point, slow has traveled `m + (k - m mod k)` total steps. Reset slow to head. Both pointers now advance at speed 1. After `m` more steps, slow reaches the cycle entry. Fast, starting from the meeting point inside the cycle, also travels `m` steps and arrives at the cycle entry — because the meeting point is exactly `m mod k` steps before the entry (going forward), and `m` is a multiple of the cycle length away from the entry plus that offset.

```math
\text{meeting point offset from entry} = k - (m \bmod k) \equiv -m \pmod{k}
```

So advancing fast by `m` from the meeting point brings it `m - m = 0` steps past the entry — exactly the entry node.

### Complexity summary

```math
\begin{array}{lcc}
\textbf{Operation} & \textbf{Singly} & \textbf{Doubly} \\
\hline
\text{Insert at head} & O(1) & O(1) \\
\text{Insert at tail (with tail ptr)} & O(1) & O(1) \\
\text{Insert at position } i & O(i) & O(i) \\
\text{Delete head} & O(1) & O(1) \\
\text{Delete tail} & O(n) & O(1) \\
\text{Delete given node} & O(n) & O(1) \\
\text{Search} & O(n) & O(n) \\
\text{Reverse} & O(n) & O(n) \\
\text{Cycle detect / find middle} & O(n) & O(n) \\
\end{array}
```

---

## Array vs. linked list comparison

| Operation | Array | Singly linked list | Notes |
| :--- | :---: | :---: | :--- |
| Random access by index | O(1) | O(n) | Arrays win decisively |
| Insert at head | O(n) shift | O(1) | LL wins |
| Insert at tail (amortized) | O(1) | O(1) w/ tail ptr | Tie |
| Insert at known pointer | O(n) shift | O(1) | LL wins |
| Delete middle (by value) | O(n) shift | O(n) walk | Tie — LL avoids the shift but still walks |
| Delete given node | O(n) | O(n) singly / O(1) doubly | DLL wins |
| Memory per element | sizeof(T) | sizeof(T) + 8 B pointer | Array wins |
| Cache locality | Excellent | Poor (pointer chase) | Arrays win in practice |
| Resize | O(n) copy | O(1) node alloc | LL wins |

> [!WARNING]
> The O(1) insert/delete advantage of linked lists is frequently overstated. Cache misses on each pointer dereference dominate real-world performance. For n < 10,000 or any read-heavy workload, a Python list (dynamic array) will outperform a hand-rolled linked list even for operations nominally favoring the linked list. Profile before reaching for a linked list.

---

## Real-world example

The canonical production use of a doubly linked list is the **LRU (Least Recently Used) cache**: a hash map for O(1) lookup combined with a doubly linked list for O(1) move-to-front (mark as recently used) and O(1) evict-least-recently-used (remove tail). This exact structure powers Python's `functools.lru_cache` and Redis's `maxmemory-policy allkeys-lru`.

```python
from __future__ import annotations
from typing import Any, Optional


class LRUNode:
    """Doubly linked list node for LRU cache."""
    __slots__ = ("key", "value", "prev", "next")

    def __init__(self, key: Any, value: Any) -> None:
        self.key: Any = key
        self.value: Any = value
        self.prev: Optional[LRUNode] = None
        self.next: Optional[LRUNode] = None


class LRUCache:
    """O(1) get and put using a hash map + doubly linked list.

    Sentinel head (most-recent end) and tail (least-recent end) eliminate
    boundary checks on every insert/delete — classic sentinel-node trick.
    """

    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.cache: dict[Any, LRUNode] = {}
        # sentinel nodes — never evicted, never returned to caller
        self.head = LRUNode(None, None)   # most-recently-used side
        self.tail = LRUNode(None, None)   # least-recently-used side
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node: LRUNode) -> None:
        """Unlink node from wherever it is. O(1) because we have prev ptr."""
        assert node.prev is not None and node.next is not None
        node.prev.next = node.next
        node.next.prev = node.prev

    def _insert_at_front(self, node: LRUNode) -> None:
        """Insert node right after the head sentinel (= most recent)."""
        node.next = self.head.next
        node.prev = self.head
        assert self.head.next is not None
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: Any) -> Any:
        """O(1). Returns the value, or -1 if not present."""
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._insert_at_front(node)
        return node.value

    def put(self, key: Any, value: Any) -> None:
        """O(1). Inserts or updates; evicts LRU entry if over capacity."""
        if key in self.cache:
            self._remove(self.cache[key])
        node = LRUNode(key, value)
        self.cache[key] = node
        self._insert_at_front(node)
        if len(self.cache) > self.capacity:
            # evict from tail side (least recently used)
            lru = self.tail.prev
            assert lru is not None and lru is not self.head
            self._remove(lru)
            del self.cache[lru.key]


# ── quick demo ────────────────────────────────────────────────────────────
if __name__ == "__main__":
    cache = LRUCache(capacity=3)
    cache.put("a", 1)
    cache.put("b", 2)
    cache.put("c", 3)
    cache.get("a")        # "a" moves to front; order: a, c, b
    cache.put("d", 4)    # evicts "b" (LRU); order: d, a, c
    assert cache.get("b") == -1
    assert cache.get("c") == 3
```

> [!TIP]
> The sentinel head/tail trick removes every `if head is None` and `if tail is None` check from `_remove` and `_insert_at_front`. With sentinels, those functions have no branches — the list is never "empty" from their perspective. This pattern appears throughout the Linux kernel's `list_head` implementation.

---

## In practice

**Python `list` is not a linked list.** `list.append` is amortized O(1) via doubling reallocation. `list.insert(0, x)` is O(n) memmove. If you need a FIFO or a deque, use `collections.deque` — it is a doubly-linked list of fixed-size blocks (not individual nodes), giving O(1) `appendleft`/`popleft` and much better cache behaviour than node-per-element designs.

**`__slots__` matters at scale.** A Python `Node` without `__slots__` carries a `__dict__` (56+ bytes overhead per instance). With `__slots__ = ("data", "next")`, CPython 3.12 packs the node into ~56 bytes total. A million-node list without `__slots__` wastes ~56 MB of avoidable overhead.

**Cycle detection in production** appears in: OS kernel process scheduling (detecting circular wait = deadlock detection), compiler intermediate representations (detecting cyclic dependencies in SSA graphs), and graph traversal (link-cut trees). Floyd's algorithm's O(1) space is the decisive advantage — a hash-set approach costs O(n) space and blows up working-set size.

**Reverse-in-groups-of-k** is a frequent interview problem with real applications: reversing segments of a doubly linked list is how `collections.deque` implements certain slice operations internally.

> [!CAUTION]
> Never call `delete_tail` in a tight loop on a singly linked list — it is O(n) per call, giving O(n²) total. If you need frequent tail deletion, use a doubly linked list (O(1) per call) or restructure as a stack (O(1) pop from head).

---

## Exercises

Each exercise includes a problem statement, a hint callout, the full Python solution with type annotations, and asymptotic complexity.

### Ex 1 — Reverse a linked list (iterative + recursive)

Reverse the node order of a singly linked list in-place. Return the new head.

> [!TIP]
> Iterative: three-pointer (prev, curr, next_node). Recursive: reverse the tail, then wire the old head to the end of the reversed tail.

```python
def reverse_iterative(head: Optional[Node]) -> Optional[Node]:
    prev: Optional[Node] = None
    curr = head
    while curr is not None:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    return prev   # new head


def reverse_recursive(head: Optional[Node]) -> Optional[Node]:
    if head is None or head.next is None:
        return head
    new_head = reverse_recursive(head.next)
    head.next.next = head   # the node after head now points back to head
    head.next = None
    return new_head
# O(n) time, O(1) space iterative; O(n) space recursive (call stack)
```

### Ex 2 — Detect a cycle

Return `True` if the linked list contains a cycle.

> [!TIP]
> Floyd's: slow moves 1, fast moves 2. If they meet, there is a cycle. If fast reaches `None`, no cycle.

```python
def detect_cycle(head: Optional[Node]) -> bool:
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
        if slow is fast:
            return True
    return False
# O(n) time, O(1) space
```

### Ex 3 — Find the start of the cycle

Return the node where the cycle begins, or `None`.

> [!TIP]
> After slow and fast meet (phase 1), reset slow to head. Advance both at speed 1. They will meet at the cycle entry (see Math section for proof).

```python
def cycle_start(head: Optional[Node]) -> Optional[Node]:
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
        if slow is fast:
            slow = head
            while slow is not fast:
                slow = slow.next  # type: ignore[union-attr]
                fast = fast.next  # type: ignore[union-attr]
            return slow
    return None
# O(n) time, O(1) space
```

### Ex 4 — Find the middle node

Return the middle node (second middle for even-length lists).

> [!TIP]
> Slow/fast: when fast hits `None`, slow is at the middle. No need to count the list length first.

```python
def middle_node(head: Optional[Node]) -> Optional[Node]:
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
    return slow
# O(n) time, O(1) space
```

### Ex 5 — Merge two sorted linked lists

Given two sorted singly linked lists, return the head of the merged sorted list. Do not allocate new nodes.

> [!TIP]
> Use a dummy sentinel head to avoid the "empty result" edge case. Compare the two current nodes and wire the smaller one into the result.

```python
def merge_sorted(
    l1: Optional[Node], l2: Optional[Node]
) -> Optional[Node]:
    dummy = Node(0)
    curr = dummy
    while l1 is not None and l2 is not None:
        if l1.data <= l2.data:
            curr.next = l1
            l1 = l1.next
        else:
            curr.next = l2
            l2 = l2.next
        curr = curr.next
    curr.next = l1 if l1 is not None else l2
    return dummy.next
# O(m + n) time, O(1) space
```

### Ex 6 — Remove the n-th node from the end

Remove the n-th node from the end of the list. Return the new head.

> [!TIP]
> Two-pointer: advance `fast` by n steps first, then move both `slow` and `fast` together until `fast` hits `None`. `slow` is then at the predecessor of the target node.

```python
def remove_nth_from_end(head: Optional[Node], n: int) -> Optional[Node]:
    dummy = Node(0)
    dummy.next = head
    slow: Optional[Node] = dummy
    fast: Optional[Node] = dummy
    for _ in range(n + 1):          # advance fast n+1 steps
        if fast is None:
            raise ValueError("n exceeds list length")
        fast = fast.next
    while fast is not None:
        slow = slow.next            # type: ignore[union-attr]
        fast = fast.next
    assert slow is not None and slow.next is not None
    slow.next = slow.next.next      # skip the target
    return dummy.next
# O(n) time, O(1) space — single pass
```

### Ex 7 — Palindrome linked list

Return `True` if the linked list reads the same forwards and backwards.

> [!TIP]
> Find middle, reverse the second half in-place, compare the two halves element by element, then optionally restore the list.

```python
def is_palindrome(head: Optional[Node]) -> bool:
    if head is None or head.next is None:
        return True
    # find middle
    slow = fast = head
    while fast is not None and fast.next is not None:
        slow = slow.next          # type: ignore[union-attr]
        fast = fast.next.next
    # reverse second half in-place
    second = reverse_iterative(slow)
    first: Optional[Node] = head
    compare = second
    result = True
    while compare is not None:
        if first is None or first.data != compare.data:
            result = False
            break
        first = first.next
        compare = compare.next
    # restore (optional but clean)
    reverse_iterative(second)
    return result
# O(n) time, O(1) space
```

### Ex 8 — Intersection of two linked lists

Given two singly linked lists that may share a suffix, return the node at which they intersect, or `None`.

> [!TIP]
> Walk both lists to their ends to get lengths `len_a` and `len_b`. Advance the longer-list pointer by `abs(len_a - len_b)`. Then walk both together until they point to the same node object (by identity, not value).

```python
def get_intersection(
    head_a: Optional[Node], head_b: Optional[Node]
) -> Optional[Node]:
    def _length(node: Optional[Node]) -> int:
        count = 0
        while node is not None:
            count += 1
            node = node.next
        return count

    len_a, len_b = _length(head_a), _length(head_b)
    a: Optional[Node] = head_a
    b: Optional[Node] = head_b
    while len_a > len_b:
        a = a.next              # type: ignore[union-attr]
        len_a -= 1
    while len_b > len_a:
        b = b.next              # type: ignore[union-attr]
        len_b -= 1
    while a is not b:
        a = a.next              # type: ignore[union-attr]
        b = b.next              # type: ignore[union-attr]
    return a   # None if no intersection
# O(m + n) time, O(1) space
```

### Ex 9 — Reverse in groups of k

Reverse the nodes of a linked list k at a time, leaving any remainder group (length < k) in its original order.

> [!TIP]
> Collect k nodes into a group. Reverse the group by relinking. Recursively process the rest. A dummy sentinel head avoids the "first group" special case.

```python
def reverse_k_group(head: Optional[Node], k: int) -> Optional[Node]:
    # check if at least k nodes remain
    curr: Optional[Node] = head
    count = 0
    while curr is not None and count < k:
        curr = curr.next
        count += 1
    if count < k:
        return head     # fewer than k nodes left — leave as-is

    # reverse exactly k nodes starting from head
    prev: Optional[Node] = None
    curr = head
    for _ in range(k):
        assert curr is not None
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    # head is now the tail of the reversed group
    # curr is the start of the next group
    assert head is not None
    head.next = reverse_k_group(curr, k)
    return prev   # prev is the new head of this reversed group
# O(n) time, O(n/k) space (recursion depth)
```

### Ex 10 — Sort a linked list (merge sort)

Sort a singly linked list in O(n log n) time and O(log n) space.

> [!TIP]
> Linked list merge sort: find the middle (slow/fast), split, recurse on each half, merge. No random access needed — this is the one sort where a linked list is not disadvantaged vs. arrays (no shifting, just relinking).

```python
def sort_list(head: Optional[Node]) -> Optional[Node]:
    if head is None or head.next is None:
        return head
    # find middle and split
    slow: Optional[Node] = head
    fast: Optional[Node] = head.next   # use head.next so slow stops at first mid
    while fast is not None and fast.next is not None:
        slow = slow.next              # type: ignore[union-attr]
        fast = fast.next.next
    assert slow is not None
    second_half = slow.next
    slow.next = None                   # cut the list
    left = sort_list(head)
    right = sort_list(second_half)
    return merge_sorted(left, right)
# O(n log n) time, O(log n) space (recursion)
```

---

## Pitfalls

- **Losing the head reference** — if you reassign `head` without first saving the old value, the entire list becomes unreachable (memory leak in Python due to reference counting not collecting immediately, worse in manual-memory languages). Always keep `head` pointing to the first real node or the sentinel.

- **Off-by-one on `None` checks** — accessing `node.next.data` without first checking `node.next is not None` raises `AttributeError`. The pattern `while fast is not None and fast.next is not None` guards both the fast pointer and the node it will double-step to.

- **Mutating a list while iterating it** — deleting a node mid-traversal without saving `next` first causes the traversal to follow a now-dangling pointer. Save `next_node = curr.next` before any mutation.

- **O(n²) from nested traversal** — calling a function that walks the list (e.g., `find_length`) inside a loop that also walks the list compounds to O(n²). The two-pointer pattern exists precisely to collapse this to O(n) with a single pass.

- **Comparing nodes by value instead of identity** — intersection detection (`a is b`) must use `is`, not `==`. Two nodes at different addresses can hold the same data without being the same node.

- **Forgetting the `prev` pointer in doubly linked deletion** — skipping `node.next.prev = node.prev` leaves a dangling back-pointer that corrupts reverse traversal silently. Update both directions every time.

- **Circular list treated as singly linked** — calling `while curr is not None` on a circular list loops forever. Termination condition must be `while curr is not self.head` (after the first iteration).

> [!CAUTION]
> In a language with manual memory management (C/C++), losing the `head` reference or deleting a node mid-traversal without saving `next` is not just a logic bug — it is a memory leak or a use-after-free. Python's garbage collector rescues you from the leak, but not from the logic corruption.

---

## Sources

- CLRS 4th ed., Chapter 10.2 — Linked Lists
- Cormen et al., *Introduction to Algorithms*, 4th edition (MIT Press, 2022)
- Floyd, R.W. (1967). "Nondeterministic algorithms." *Journal of the ACM* 14(4): 636–644. — original tortoise-and-hare publication
- CPython source: `Objects/listobject.c` (dynamic array), `Modules/_collectionsmodule.c` (deque as block-linked-list)
- Linux kernel `include/linux/list.h` — intrusive doubly linked list with sentinel pattern
- Python docs: [`collections.deque`](https://docs.python.org/3/library/collections.html#collections.deque)
- Conversation with user on 2026-05-19

---

## Related

- [[02-arrays-and-searching]]
- [[07-stacks]]
- [[08-queues]]
- [[09-trees]]
