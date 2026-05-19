---
title: "Queues"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# Queues

[toc]

> **TL;DR:** A queue is a FIFO abstract data type — the element inserted first is the element removed first. It models any real-world waiting line: network packets, OS process schedulers, BFS frontiers, message brokers. The implementation challenge is doing enqueue and dequeue in O(1) without wasting memory — the circular-buffer trick solves the array case, and `collections.deque` solves it for Python. Priority queues break strict FIFO in favor of ordering by priority, typically backed by a binary min-heap.

## Vocabulary

- **Queue** — an ADT with FIFO ordering: the first item enqueued is the first dequeued.
- **FIFO** — First In, First Out. The invariant that defines a queue. Contrast with LIFO (stack).
- **Front** — the end from which elements are dequeued (removed). The "head" of the line.
- **Rear** — the end at which elements are enqueued (inserted). Also called "back" or "tail".
- **Enqueue** — insert an element at the rear.
- **Dequeue** — remove and return the element at the front.
- **Circular queue** — array-backed queue where front and rear indices wrap around using modular arithmetic, eliminating the wasted-space problem of a naive array queue.
- **Deque** — Double-Ended Queue. Supports enqueue and dequeue at *both* ends in O(1). Python's `collections.deque` is a doubly linked list of fixed-size blocks.
- **Priority queue** — a queue where each element carries a priority; dequeue always returns the highest-priority element regardless of insertion order.
- **Min-heap** — a complete binary tree satisfying the heap property: every node's key is ≤ its children's keys. Used to back a priority queue with O(log n) push/pop.
- **Max-heap** — heap where every node's key is ≥ its children's. Python's `heapq` is a min-heap; negate keys to simulate max-heap.
- **Monotonic deque** — a deque maintained in strictly increasing or decreasing order, discarding elements that can never be the answer; the engine behind O(n) sliding-window-maximum.

## Intuition

Picture a single-file ticket queue at a cinema. You join at the back (enqueue at rear), get served from the front (dequeue at front). No cutting. The cinema only cares about two pointers: "who is next?" (front) and "where does the next arrival stand?" (rear). The middle is opaque — you never touch it.

The implementation tension is that arrays are fast but statically indexed. After dequeueing N items the front pointer drifts right, leaving N wasted cells at the left. The circular queue insight: treat the array as a ring — wrap the rear back to index 0 when it falls off the right edge. Now the array is never wasted. For variable-size queues, a linked list removes the capacity ceiling entirely, at the cost of pointer overhead and cache mislocality.

Priority queues break FIFO to serve the "most important" first. The heap provides the right trade-off: O(1) peek at the minimum, O(log n) insert and remove. This is covered in depth in [[09-trees]].

## Math foundations

The queue ADT sits atop several non-trivial mathematical results that are worth understanding in their own right. Modular arithmetic makes the circular buffer work without branching. Amortized analysis explains why the two-stacks queue is genuinely O(1) per operation despite occasional O(n) spikes. Heap-shape geometry explains O(log n) push/pop and the surprising O(n) build. Little's law ties all of this to real-world system design, connecting data-structure theory to production queue sizing.

### Modular arithmetic for the circular buffer

A circular buffer of capacity m reuses its array slots by wrapping indices with the modulo operator. Every enqueue advances the rear pointer and every dequeue advances the front pointer, both wrapping at m. The number of stored items is recovered from the two pointers alone, but the naive formula `rear - front` goes negative when rear has wrapped around past front — the `+ m` term corrects this.

```math
\text{rear} \leftarrow (\text{rear} + 1) \bmod m
```

```math
\text{front} \leftarrow (\text{front} + 1) \bmod m
```

```math
\text{size} = (\text{rear} - \text{front} + m) \bmod m
```

The `+ m` before the outer `mod` is the critical detail: in any language where the `%` operator can return a negative value for negative operands (C, C++, Java, Go), `(-1 + m) % m = m - 1` is correct, whereas `(-1) % m` may return `-1`. Python's `%` always returns a non-negative result for positive m, but the `+ m` convention is universal and makes the intent explicit regardless of language.

> [!IMPORTANT]
> The size formula `(rear - front + m) % m` returns 0 when the buffer is **either** empty or completely full (both cases have `front == rear`). You must track size with a separate counter — or sacrifice one slot and treat `(rear + 1) % m == front` as the full condition — to distinguish the two states.

### Amortized O(1) for two-stacks queue

The two-stacks queue (inbox + outbox) performs O(1) amortized dequeue even though a single dequeue can cost O(n) in the worst case. The aggregate method of amortized analysis counts total work across a sequence of n operations rather than worst-casing each one individually. Each element is pushed onto inbox exactly once (cost 1) and transferred from inbox to outbox at most once (cost 1), giving a total transfer cost of at most n across all n dequeue calls. Spreading that cost over n operations yields O(1) per operation.

```math
\begin{aligned}
&\text{Cost of } n \text{ enqueues: each element pushed to inbox once} \Rightarrow O(n) \\
&\text{Cost of } n \text{ dequeues: each element transferred inbox→outbox at most once} \Rightarrow O(n) \\
&\text{Total work for } n \text{ operations: } O(n) + O(n) = O(n) \\
&\text{Amortized cost per operation: } \frac{O(n)}{n} = O(1)
\end{aligned}
```

The key precondition is that transfers happen lazily — only when outbox is empty — so no element crosses the inbox→outbox boundary more than once. Any scheme that eagerly re-transfers elements would break the O(1) amortized guarantee.

> [!WARNING]
> Worst-case dequeue remains O(n). In a real-time system with hard latency deadlines (audio DSP, control loops, HFT), O(1) amortized is not sufficient — use a true O(1) worst-case queue (ring buffer with fixed capacity, or a lock-free SPSC queue). Amortized guarantees are averages; individual operations can spike.

### Priority queue math — heap invariant

A binary min-heap is a complete binary tree stored implicitly as a flat array. The implicit indexing formula is the key: no pointers needed, children and parent are computed arithmetically. The heap property — every parent is smaller than or equal to both children — ensures the minimum is always at index 0 (the root), reachable in O(1).

```math
\text{parent}(i) = \left\lfloor \frac{i - 1}{2} \right\rfloor
```

```math
\text{left child}(i) = 2i + 1, \qquad \text{right child}(i) = 2i + 2
```

```math
\text{Heap property (min-heap): } A[\text{parent}(i)] \leq A[i] \quad \forall\, i > 0
```

The height of a heap with n nodes follows directly from the completeness constraint — all levels are full except possibly the last, which is left-filled. Counting levels gives:

```math
h = \lfloor \log_2 n \rfloor
```

Because sift-up (heappush) and sift-down (heappop) each traverse at most h levels, both operations are O(log n). The heap's implicit array layout also makes it cache-friendly — no pointer chasing, sequential access on each level, predictable prefetching.

### Build-heap is O(n), not O(n log n)

Inserting n elements one by one into a heap via repeated heappush costs O(n log n). But bottom-up heapify — the algorithm behind `heapq.heapify` — is O(n). The difference is in how much sifting each node requires: leaf nodes (the bottom half of the array) need zero sifting; nodes near the root need at most O(log n). Summing over all nodes with the geometric series identity gives O(n) total.

```math
\sum_{h=0}^{\lfloor \log_2 n \rfloor} \left\lceil \frac{n}{2^{h+1}} \right\rceil \cdot h
\;\leq\; n \sum_{h=0}^{\infty} \frac{h}{2^{h+1}}
\;=\; n \cdot \frac{1/2}{(1 - 1/2)^2}
\;=\; n \cdot 2
\;=\; O(n)
```

The identity used is the derivative of the geometric series: for |x| < 1,

```math
\sum_{h=0}^{\infty} h \cdot x^h = \frac{x}{(1-x)^2}
```

evaluated at x = 1/2, giving 2. The intuition is that most of the n/2 leaf nodes have height 0 and do no work; the single root node has height ⌊log n⌋ but there is only one of it. Work is concentrated at the bottom, not the top.

> [!NOTE]
> This is why `heapq.heapify(lst)` is O(n) while n calls to `heapq.heappush` are O(n log n). For building a priority queue from a pre-existing list, always prefer `heapify`. The Heapsort algorithm must still pay O(n log n) for the sort phase, but the heap-build phase is O(n).

### Little's law (queueing theory preview)

Little's law is a fundamental result in queueing theory that relates the three key observables of any stable queue system: the average number of items in the system (L), the arrival rate (λ), and the average time each item spends in the system (W). It holds for any queue discipline — FIFO, LIFO, priority, random — under the sole condition that the system is stable (arrival rate does not exceed service rate).

```math
L = \lambda \cdot W
```

A concrete example: if a web service receives 100 requests per second (λ = 100 req/s) and each request takes an average of 50 ms to complete (W = 0.05 s), then on average there are L = 100 × 0.05 = 5 requests in flight at any moment. This directly determines the minimum queue buffer size needed to absorb bursts and the latency cost of increasing throughput.

```math
\text{Example: } \lambda = 100\,\text{req/s},\; W = 50\,\text{ms} = 0.05\,\text{s} \;\Longrightarrow\; L = 100 \times 0.05 = 5\,\text{in flight}
```

Little's law is also a diagnostic tool: if you measure L and λ in production (both are easy to instrument), you can infer W without tracing individual requests. Conversely, if latency W grows while L is bounded, λ must be falling — a sign of a capacity bottleneck upstream.

> [!TIP]
> Little's law underpins every queueing-theory-based capacity model: service mesh backpressure limits, Kafka consumer lag targets, database connection pool sizing, and OS I/O queue depth tuning. Whenever you see a bounded queue filling up, the question is: is W growing (latency spike), or is λ growing (traffic surge)? L = λW lets you separate the two.


## Memory layout

The circular queue is the central visualization for this topic. Three frames show the state machine across enqueue and dequeue operations.

**Figure:** Circular queue — a 6-slot array with front and rear pointers wrapping around.

```
Initial state (empty, capacity = 6)
  Index:  0    1    2    3    4    5
         ┌────┬────┬────┬────┬────┬────┐
         │    │    │    │    │    │    │
         └────┴────┴────┴────┴────┴────┘
          ↑
        front = rear = 0    (size = 0)

After enqueue(10), enqueue(20), enqueue(30)
  Index:  0    1    2    3    4    5
         ┌────┬────┬────┬────┬────┬────┐
         │ 10 │ 20 │ 30 │    │    │    │
         └────┴────┴────┴────┴────┴────┘
          ↑              ↑
        front=0        rear=3    (size = 3)

After dequeue() → 10, then enqueue(40), enqueue(50), enqueue(60)
  Index:  0    1    2    3    4    5
         ┌────┬────┬────┬────┬────┬────┐
         │    │ 20 │ 30 │ 40 │ 50 │ 60 │
         └────┴────┴────┴────┴────┴────┘
               ↑                        ↑
             front=1               rear=0 (wrapped!)

  rear = (5 + 1) % 6 = 0  ← modular arithmetic does the wrap

After enqueue(70) — fills the one remaining slot at index 0:
  Index:  0    1    2    3    4    5
         ┌────┬────┬────┬────┬────┬────┐
         │ 70 │ 20 │ 30 │ 40 │ 50 │ 60 │
         └────┴────┴────┴────┴────┴────┘
          ↑     ↑
        rear=0  front=1     FULL: size == capacity

  Full check: (rear + 1) % capacity == front
```

The key invariant: `rear = (rear + 1) % capacity` on every enqueue, `front = (front + 1) % capacity` on every dequeue. Full when `size == capacity` (track size separately, or sacrifice one slot to distinguish full from empty).

## How it works

A queue's six operations form the complete ADT contract. Knowing their signatures and their complexity for each implementation is the prerequisite for choosing the right backing structure.

| Operation | Description | Array (circular) | Linked list | `collections.deque` |
| :--- | :--- | :---: | :---: | :---: |
| `enqueue(x)` | Insert x at rear | O(1) amortized | O(1) | O(1) |
| `dequeue()` | Remove + return front | O(1) | O(1) | O(1) |
| `front()` | Peek front without removing | O(1) | O(1) | O(1) |
| `rear()` | Peek rear without removing | O(1) | O(1) | O(1) |
| `is_empty()` | True if size == 0 | O(1) | O(1) | O(1) |
| `size()` | Number of elements | O(1) | O(1) | O(1) |

### Array queue and its wasted-space problem

The naive array queue keeps a `front` index and a `rear` index. Every dequeue increments `front` rightward, leaving the vacated slot inaccessible. After n dequeues on a capacity-n array the queue appears full but is logically empty — every slot to the left of `front` is wasted. The only fix without the circular trick is to shift all remaining elements left after each dequeue, costing O(n) per dequeue. This is exactly the O(n) penalty you pay with Python's `list.pop(0)`.

```python
from typing import Any


class NaiveArrayQueue:
    """Linear array queue — illustrative only. dequeue is O(n)."""

    def __init__(self) -> None:
        self._data: list[Any] = []

    def enqueue(self, x: Any) -> None:
        self._data.append(x)          # O(1) amortized

    def dequeue(self) -> Any:
        if not self._data:
            raise IndexError("dequeue from empty queue")
        return self._data.pop(0)      # O(n) — shifts every element left

    def front(self) -> Any:
        if not self._data:
            raise IndexError("front of empty queue")
        return self._data[0]

    def is_empty(self) -> bool:
        return len(self._data) == 0

    def size(self) -> int:
        return len(self._data)
```

> [!WARNING]
> `list.pop(0)` is O(n) because CPython's list is backed by a dynamic array — removing index 0 shifts every subsequent pointer one slot left. At n = 100,000 elements this makes BFS O(n²). Never use a plain `list` as a FIFO queue in production code.

### Circular queue

The circular queue wraps rear back to index 0 via modular arithmetic, recycling freed slots. Front and rear chase each other around the ring. A separate `_size` counter distinguishes the full state (size == capacity) from the empty state (size == 0), avoiding the need to sacrifice one slot.

```python
from typing import Any


class CircularQueue:
    """Fixed-capacity circular queue. All operations O(1)."""

    def __init__(self, capacity: int) -> None:
        self._capacity = capacity
        self._data: list[Any] = [None] * capacity
        self._front: int = 0
        self._rear: int = 0     # points to the next write slot
        self._size: int = 0

    def enqueue(self, x: Any) -> None:
        if self._size == self._capacity:
            raise OverflowError("queue is full")
        self._data[self._rear] = x
        self._rear = (self._rear + 1) % self._capacity
        self._size += 1

    def dequeue(self) -> Any:
        if self._size == 0:
            raise IndexError("dequeue from empty queue")
        val = self._data[self._front]
        self._data[self._front] = None      # allow GC
        self._front = (self._front + 1) % self._capacity
        self._size -= 1
        return val

    def front(self) -> Any:
        if self._size == 0:
            raise IndexError("front of empty queue")
        return self._data[self._front]

    def rear(self) -> Any:
        if self._size == 0:
            raise IndexError("rear of empty queue")
        return self._data[(self._rear - 1) % self._capacity]

    def is_empty(self) -> bool:
        return self._size == 0

    def size(self) -> int:
        return self._size
```

> [!IMPORTANT]
> The full condition is `self._size == self._capacity`, not `self._rear == self._front`. Those two indices are equal in both the full and the empty state — you cannot distinguish them without a separate counter.

### Linked-list queue

A singly linked list with two pointers — `_head` at the front and `_tail` at the rear — gives O(1) enqueue and O(1) dequeue with no capacity limit. The tradeoff is one pointer's worth of overhead per node (8 bytes on a 64-bit system) plus cache misses when traversing nodes that are not laid out contiguously in memory.

```python
from __future__ import annotations
from typing import Any, Optional


class _Node:
    __slots__ = ("val", "next")

    def __init__(self, val: Any) -> None:
        self.val = val
        self.next: Optional[_Node] = None


class LinkedQueue:
    """Singly linked list queue. enqueue/dequeue both O(1)."""

    def __init__(self) -> None:
        self._head: Optional[_Node] = None   # front
        self._tail: Optional[_Node] = None   # rear
        self._size: int = 0

    def enqueue(self, x: Any) -> None:
        node = _Node(x)
        if self._tail is None:
            self._head = self._tail = node
        else:
            self._tail.next = node
            self._tail = node
        self._size += 1

    def dequeue(self) -> Any:
        if self._head is None:
            raise IndexError("dequeue from empty queue")
        val = self._head.val
        self._head = self._head.next
        if self._head is None:
            self._tail = None
        self._size -= 1
        return val

    def front(self) -> Any:
        if self._head is None:
            raise IndexError("front of empty queue")
        return self._head.val

    def is_empty(self) -> bool:
        return self._size == 0

    def size(self) -> int:
        return self._size
```

### Deque (double-ended queue)

A deque generalises the queue by allowing O(1) insertion and removal at *both* ends. This makes it simultaneously a queue, a stack, and the engine for the monotonic-deque pattern. Python's `collections.deque` is the canonical implementation: internally it is a doubly linked list of fixed-size blocks (each block holds 64 `PyObject*` pointers on CPython), so indexing into the middle is O(n) but head/tail operations are O(1) and cache-friendlier than a naive linked list.

```python
from collections import deque

dq: deque[int] = deque()

# Deque as a queue (FIFO)
dq.append(10)          # enqueue at right (rear)
dq.append(20)
dq.append(30)
print(dq.popleft())    # dequeue from left (front) → 10

# Deque as a stack (LIFO)
dq.appendleft(99)      # push to front
print(dq.popleft())    # pop from front → 99

# Deque as a sliding window buffer (maxlen)
window: deque[int] = deque(maxlen=3)
for x in [1, 2, 3, 4, 5]:
    window.append(x)   # auto-discards oldest when full
print(list(window))    # [3, 4, 5]
```

### Priority queue

A priority queue abandons FIFO in favor of returning the element with the highest priority on each dequeue. The canonical backing structure is a binary min-heap: a complete binary tree stored implicitly in an array where the root is always the minimum. Python's `heapq` module exposes a min-heap over a plain `list`. To simulate a max-heap, negate all keys before pushing.

```python
import heapq

# Min-heap (default in Python)
min_heap: list[int] = []
heapq.heappush(min_heap, 5)
heapq.heappush(min_heap, 1)
heapq.heappush(min_heap, 3)
print(heapq.heappop(min_heap))   # → 1  (smallest)

# Max-heap via negation
max_heap: list[int] = []
for x in [5, 1, 3]:
    heapq.heappush(max_heap, -x)
print(-heapq.heappop(max_heap))  # → 5  (largest)

# Heap with (priority, item) tuples
task_heap: list[tuple[int, str]] = []
heapq.heappush(task_heap, (2, "render"))
heapq.heappush(task_heap, (1, "auth"))
heapq.heappush(task_heap, (3, "log"))
priority, task = heapq.heappop(task_heap)
print(task)   # → "auth"  (lowest priority number = highest urgency)
```

> [!NOTE]
> For a full treatment of the binary heap's internal structure, sift-up/sift-down, and heap sort, see [[09-trees]]. Here we treat the heap as a black box and focus on the queue ADT it enables.

## Math

The amortized complexity of `collections.deque` and the circular queue both deserve formal treatment, because the raw worst-case numbers can mislead.

**Circular queue** — all six operations are strictly O(1) when capacity is known at construction. No amortization needed; modular arithmetic is a single integer instruction.

**`collections.deque` (dynamic)** — when the deque needs to grow, it allocates a new block (fixed size, currently 64 pointers on CPython). The cost of block allocation is amortized over all the appends that fill that block. The amortized cost per append is:

```math
T_{\text{amortized}} = \frac{\text{total allocation cost over } n \text{ appends}}{n}
                     = \frac{O(n/B) \cdot O(B)}{n}
                     = O(1)
```

where B = 64 is the block size. Unlike a Python `list`, a `deque` never copies existing elements on growth — it adds a new block. So `deque.appendleft` and `deque.append` are both O(1) amortized, with no worst-case O(n) spike.

**Monotonic deque (sliding window maximum)** — each element is pushed onto the deque exactly once and popped at most once, giving O(n) total work for a window of size k across an array of size n. The number of output windows is:

```math
\text{num\_windows} = n - k + 1
```

**Priority queue (binary heap)** — heap operations:

```math
\text{heappush: } O(\log n) \qquad \text{heappop: } O(\log n) \qquad \text{heapify (build): } O(n)
```

The O(n) build via `heapq.heapify` is non-obvious: bottom-up construction does less sifting than n individual pushes would, because most nodes are near the leaves.

## Real-world example

Production systems use queues at every layer of the stack. Here are four concrete patterns, each with runnable code.

### BFS frontier

Breadth-first search over a graph uses a queue as its frontier. Every node dequeued spawns its unvisited neighbours enqueued — FIFO order guarantees the shortest-path property on unweighted graphs.

```python
from collections import deque
from typing import Optional


def bfs_shortest_path(
    graph: dict[int, list[int]],
    source: int,
    target: int,
) -> Optional[list[int]]:
    """Return shortest path from source to target, or None if unreachable."""
    visited: set[int] = {source}
    parent: dict[int, Optional[int]] = {source: None}
    queue: deque[int] = deque([source])

    while queue:
        node = queue.popleft()          # O(1) dequeue
        if node == target:
            path: list[int] = []
            cur: Optional[int] = target
            while cur is not None:
                path.append(cur)
                cur = parent[cur]
            return path[::-1]
        for neighbour in graph[node]:
            if neighbour not in visited:
                visited.add(neighbour)
                parent[neighbour] = node
                queue.append(neighbour)  # O(1) enqueue

    return None


if __name__ == "__main__":
    g = {0: [1, 2], 1: [3], 2: [3, 4], 3: [5], 4: [5], 5: []}
    print(bfs_shortest_path(g, 0, 5))   # [0, 2, 4, 5] or [0, 1, 3, 5]
```

> [!TIP]
> Always use `collections.deque` for BFS, never `list`. On a graph with 1M nodes, `list.pop(0)` degrades BFS from O(V+E) to O(V²+E). This is one of the most common competitive-programming performance bugs.

### Thread-safe task queue

`queue.Queue` is Python's thread-safe FIFO queue, backed by a `collections.deque` under a lock. It is the canonical producer-consumer channel in CPython's threading model.

```python
import queue
import threading
import time


def worker(task_queue: "queue.Queue[str]") -> None:
    while True:
        task = task_queue.get()         # blocks until an item is available
        if task == "__STOP__":
            task_queue.task_done()
            break
        print(f"[worker] processing: {task}")
        time.sleep(0.05)                # simulate work
        task_queue.task_done()


def main() -> None:
    q: queue.Queue[str] = queue.Queue(maxsize=10)
    t = threading.Thread(target=worker, args=(q,), daemon=True)
    t.start()

    for job in ["render", "compress", "upload", "notify"]:
        q.put(job)
    q.put("__STOP__")
    q.join()   # blocks until all task_done() calls match put() calls
    t.join()


if __name__ == "__main__":
    main()
```

### Async queue

For async/await code, `asyncio.Queue` provides the same producer-consumer contract without threads — ideal for I/O-bound pipelines.

```python
import asyncio


async def producer(q: asyncio.Queue[int]) -> None:
    for i in range(5):
        await q.put(i)
        await asyncio.sleep(0.01)
    await q.put(-1)   # sentinel


async def consumer(q: asyncio.Queue[int]) -> None:
    while True:
        item = await q.get()
        if item == -1:
            q.task_done()
            break
        print(f"consumed: {item}")
        q.task_done()


async def main() -> None:
    q: asyncio.Queue[int] = asyncio.Queue()
    await asyncio.gather(producer(q), consumer(q))


asyncio.run(main())
```

### Task scheduler (priority queue)

An OS-style task scheduler dispatches the highest-priority runnable task. `heapq` gives O(log n) dispatch with Python's built-in list.

```python
import heapq
from dataclasses import dataclass, field
from typing import Any


@dataclass(order=True)
class Task:
    priority: int
    name: str = field(compare=False)


def run_scheduler(tasks: list[Task]) -> None:
    heap: list[Task] = []
    for task in tasks:
        heapq.heappush(heap, task)

    while heap:
        task = heapq.heappop(heap)
        print(f"[scheduler] running priority={task.priority}: {task.name}")


if __name__ == "__main__":
    run_scheduler([
        Task(3, "background-sync"),
        Task(1, "interrupt-handler"),
        Task(2, "user-event"),
        Task(1, "network-recv"),
    ])
    # Output order: interrupt-handler, network-recv, user-event, background-sync
```

## Two-stacks queue / two-queues stack

These are classic interview constructs that reveal how FIFO and LIFO relate. The key insight is that two reversals produce the original order — so a second stack "unreverses" a first stack, recovering FIFO.

### Queue using two stacks

Maintain an `inbox` stack for enqueue and an `outbox` stack for dequeue. When `outbox` is empty, pour all of `inbox` into `outbox`, reversing order. The amortized cost per operation is O(1): each element crosses from inbox to outbox exactly once.

```python
from typing import Any


class QueueViaStacks:
    """FIFO queue backed by two LIFO stacks. Amortized O(1) per operation."""

    def __init__(self) -> None:
        self._inbox: list[Any] = []
        self._outbox: list[Any] = []

    def enqueue(self, x: Any) -> None:
        self._inbox.append(x)

    def _transfer(self) -> None:
        if not self._outbox:
            while self._inbox:
                self._outbox.append(self._inbox.pop())

    def dequeue(self) -> Any:
        self._transfer()
        if not self._outbox:
            raise IndexError("dequeue from empty queue")
        return self._outbox.pop()

    def front(self) -> Any:
        self._transfer()
        if not self._outbox:
            raise IndexError("front of empty queue")
        return self._outbox[-1]

    def is_empty(self) -> bool:
        return not self._inbox and not self._outbox
```

### Stack using two queues

Push to `q1`; pop requires rotating all but the last element through `q2` then swapping. Pop is O(n). There is no O(1) amortized trick here — one direction must pay.

```python
from collections import deque
from typing import Any


class StackViaQueues:
    """LIFO stack backed by two FIFO queues. push O(1), pop O(n)."""

    def __init__(self) -> None:
        self._q1: deque[Any] = deque()
        self._q2: deque[Any] = deque()

    def push(self, x: Any) -> None:
        self._q1.append(x)

    def pop(self) -> Any:
        if not self._q1:
            raise IndexError("pop from empty stack")
        # Move all but last element to q2
        while len(self._q1) > 1:
            self._q2.append(self._q1.popleft())
        val = self._q1.popleft()          # the last (top-most) element
        self._q1, self._q2 = self._q2, self._q1   # swap roles
        return val

    def top(self) -> Any:
        if not self._q1:
            raise IndexError("top of empty stack")
        # Rotate all to q2, save last, restore
        while len(self._q1) > 1:
            self._q2.append(self._q1.popleft())
        val = self._q1.popleft()
        self._q2.append(val)
        self._q1, self._q2 = self._q2, self._q1
        return val

    def is_empty(self) -> bool:
        return not self._q1
```

## In practice

At the single-process level, `collections.deque` is the right default for any Python FIFO. It is a C-implemented doubly linked list of blocks, thread-unsafe by design (use `queue.Queue` instead when crossing thread boundaries). At scale, the queue abstraction is implemented by dedicated brokers:

- **Redis** — `LPUSH` / `BRPOP` as a blocking queue; `ZADD` / `ZPOPMIN` as a priority queue. Single-threaded command execution makes operations atomic without locks.
- **Kafka** — distributed, durable, partitioned append-only log. Consumers track their own offset. Not a true queue (no dequeue semantics) but models FIFO within a partition.
- **RabbitMQ** — AMQP broker with true queue semantics, dead-letter exchanges, TTL, and priority queues.

In the OS: the Linux `kfifo` is a lock-free single-producer single-consumer circular buffer in the kernel. The CPU's hardware interrupt controller maintains a priority queue of pending interrupts (IRQ levels).

> [!TIP]
> Choosing the right Python queue class by context:
> - Single-process, single-thread: `collections.deque`
> - Multi-thread producer/consumer: `queue.Queue` (GIL-safe, blocking `get`)
> - Async coroutines: `asyncio.Queue`
> - Process-crossing: `multiprocessing.Queue`
> - Priority dispatch: `heapq` over a `list` (not thread-safe; wrap with a lock if shared)

> [!CAUTION]
> `heapq` with mutable priority keys is broken. If you push `(priority, item)` and later change an item's priority by mutating the tuple, the heap property is violated silently. The standard fix is to push a "tombstone" entry (mark the old entry as invalid with a removed flag) and push a fresh entry with the updated priority — then filter tombstones on pop. See the Python docs on "heap entries with a deletion counter" for the canonical pattern.

## Pitfalls

- **Using `list.pop(0)` as dequeue** — this compiles, runs, and produces correct results. It is also O(n) per call. An O(n log n) sort followed by O(n²) dequeues is worse than a naive O(n²) approach on large inputs. Use `collections.deque.popleft()` instead.

- **Off-by-one in circular index wrap** — the condition `rear == capacity` (no modulo) misses the wrap. The correct update is `rear = (rear + 1) % capacity`. Failing to use modulo means the array index goes out of bounds or you lose the circular property entirely.

- **Confusing full and empty in a circular queue** — when `front == rear` you cannot tell if the queue is full or empty without additional state. Solutions: track a `_size` counter (clean), or sacrifice one slot so that full means `(rear + 1) % capacity == front` and empty means `front == rear`.

- **Assuming `queue.Queue` is faster than `deque`** — `queue.Queue` wraps a `deque` with a `threading.Condition` lock. In a single-threaded context it is strictly slower than a bare `deque`. Only use it when you need blocking `get()` semantics across threads.

- **Inverting heap polarity incorrectly** — to get a max-heap with tuple keys `(priority, item)`, negate only the priority: push `(-priority, item)`. Negating the item too (if it is a string or complex object) raises a `TypeError` when Python compares tuples with equal priorities.

- **Monotonic deque direction** — for sliding window *maximum*, maintain a *decreasing* deque (front holds the largest, remove from front when it falls out of the window). For sliding window *minimum*, maintain an *increasing* deque. Reversing the inequality gives the wrong answer silently.

> [!CAUTION]
> Modifying a `heapq` list directly (sorting it, inserting at arbitrary positions) silently destroys the heap property. Only use `heapq.heappush`, `heapq.heappop`, `heapq.heapreplace`, and `heapq.heapify` to mutate the list. Any other mutation produces incorrect `heappop` results.

## Exercises

Seven problems with full Python solutions. Problems are ordered by concept: ADT → circular → two-stacks → sliding window → heap.

---

### Exercise 1 — Queue using two stacks (LeetCode 232)

Implement a FIFO queue using only two stacks with amortized O(1) per operation.

> [!TIP]
> Maintain an `inbox` and `outbox` stack. Pour `inbox` into `outbox` lazily — only when `outbox` is empty and a dequeue is requested. Each element makes the transfer exactly once.

```python
from typing import Any


class MyQueue:
    """LeetCode 232 — Queue via Two Stacks."""

    def __init__(self) -> None:
        self._inbox: list[Any] = []
        self._outbox: list[Any] = []

    def push(self, x: Any) -> None:
        self._inbox.append(x)

    def _flush(self) -> None:
        if not self._outbox:
            while self._inbox:
                self._outbox.append(self._inbox.pop())

    def pop(self) -> Any:
        self._flush()
        return self._outbox.pop()

    def peek(self) -> Any:
        self._flush()
        return self._outbox[-1]

    def empty(self) -> bool:
        return not self._inbox and not self._outbox


# Complexity: push O(1), pop O(1) amortized, peek O(1) amortized.
```

---

### Exercise 2 — Stack using two queues (LeetCode 225)

Implement a LIFO stack using only two queues. Push must be O(1); pop is O(n).

> [!TIP]
> After pushing to `q1`, immediately rotate all elements through `q2` and swap, so the newest element is always at the front of `q1`. This makes top and pop O(1) and push O(n) instead — pick the direction that fits your access pattern.

```python
from collections import deque
from typing import Any


class MyStack:
    """LeetCode 225 — Stack via Two Queues. push O(n), pop O(1)."""

    def __init__(self) -> None:
        self._q: deque[Any] = deque()

    def push(self, x: Any) -> None:
        self._q.append(x)
        # Rotate so newest is at front
        for _ in range(len(self._q) - 1):
            self._q.append(self._q.popleft())

    def pop(self) -> Any:
        return self._q.popleft()

    def top(self) -> Any:
        return self._q[0]

    def empty(self) -> bool:
        return not self._q


# Complexity: push O(n), pop O(1), top O(1).
```

---

### Exercise 3 — Design circular queue (LeetCode 622)

Design a circular queue with fixed capacity supporting: `enQueue`, `deQueue`, `Front`, `Rear`, `isEmpty`, `isFull`.

> [!TIP]
> Track `_front`, `_rear`, and `_size` separately. Use `(self._rear + 1) % self._capacity` to advance rear on enqueue and `(self._front + 1) % self._capacity` to advance front on dequeue.

```python
class MyCircularQueue:
    """LeetCode 622 — Design Circular Queue."""

    def __init__(self, k: int) -> None:
        self._data: list[int] = [0] * k
        self._front: int = 0
        self._rear: int = 0
        self._size: int = 0
        self._capacity: int = k

    def enQueue(self, value: int) -> bool:
        if self._size == self._capacity:
            return False
        self._data[self._rear] = value
        self._rear = (self._rear + 1) % self._capacity
        self._size += 1
        return True

    def deQueue(self) -> bool:
        if self._size == 0:
            return False
        self._front = (self._front + 1) % self._capacity
        self._size -= 1
        return True

    def Front(self) -> int:
        return -1 if self._size == 0 else self._data[self._front]

    def Rear(self) -> int:
        if self._size == 0:
            return -1
        return self._data[(self._rear - 1) % self._capacity]

    def isEmpty(self) -> bool:
        return self._size == 0

    def isFull(self) -> bool:
        return self._size == self._capacity


# All operations O(1).
```

---

### Exercise 4 — Sliding window maximum (LeetCode 239)

Given array `nums` and window size `k`, return the maximum of each window of size `k`.

**Input:** `nums = [10, 8, 5, 12, 15, 7, 6]`, `k = 3`
**Output:** `[10, 12, 15, 15, 15]`

> [!TIP]
> Maintain a monotonic *decreasing* deque of **indices**. Before adding index `i`, pop from the deque's right any index whose `nums` value is less than `nums[i]` — those can never be the window maximum. Pop from the left any index that has left the window (`deque[0] <= i - k`).

```python
from collections import deque


def max_sliding_window(nums: list[int], k: int) -> list[int]:
    """
    Monotonic deque — O(n) time, O(k) space.
    Deque stores indices; front always holds index of current window max.
    """
    result: list[int] = []
    dq: deque[int] = deque()  # indices, decreasing by nums value

    for i, val in enumerate(nums):
        # Remove indices no longer in the window
        while dq and dq[0] <= i - k:
            dq.popleft()

        # Maintain decreasing order: remove smaller elements from the right
        while dq and nums[dq[-1]] < val:
            dq.pop()

        dq.append(i)

        # Window is fully formed starting at index k-1
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result


# Test
print(max_sliding_window([10, 8, 5, 12, 15, 7, 6], 3))
# → [10, 12, 15, 15, 15]
# Complexity: O(n) time — each element enqueued and dequeued at most once.
```

---

### Exercise 5 — First negative integer in every window of size k

Given array (can contain negatives), return the first negative integer in each window. If none, return 0.

> [!TIP]
> Use a deque storing indices of negative numbers only. Remove from the left when the index falls outside the window. The front of the deque is always the first negative in the current window.

```python
from collections import deque


def first_negative_in_window(nums: list[int], k: int) -> list[int]:
    """O(n) time using a deque of indices of negative elements."""
    result: list[int] = []
    dq: deque[int] = deque()  # stores indices of negative elements

    for i, val in enumerate(nums):
        if val < 0:
            dq.append(i)

        # Remove indices outside the current window
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        if i >= k - 1:
            result.append(nums[dq[0]] if dq else 0)

    return result


# Test
print(first_negative_in_window([-8, 2, 3, -6, 10], 2))
# → [-8, 0, -6, -6]
```

---

### Exercise 6 — Top-K frequent elements (LeetCode 347)

Given `nums` and integer `k`, return the `k` most frequent elements in any order.

> [!TIP]
> Count frequencies with a `Counter`, then use `heapq.nlargest` on the frequency map. Alternatively, maintain a min-heap of size `k` — when the heap exceeds `k`, pop the minimum-frequency element.

```python
import heapq
from collections import Counter


def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """
    O(n log k) time using a min-heap of size k.
    Keeps the k highest-frequency elements; discards lower ones.
    """
    freq = Counter(nums)
    # (frequency, element) min-heap — smallest frequency at top
    heap: list[tuple[int, int]] = []
    for element, count in freq.items():
        heapq.heappush(heap, (count, element))
        if len(heap) > k:
            heapq.heappop(heap)   # evict least frequent

    return [element for _, element in heap]


# Test
print(top_k_frequent([1, 1, 1, 2, 2, 3], 2))   # → [2, 1]  (or [1, 2])
# Complexity: O(n log k) time, O(n) space for frequency map.
```

---

### Exercise 7 — Task scheduler (LeetCode 621)

Given a list of CPU tasks (letters) and a cooldown `n` (same task must wait at least `n` intervals between executions), return the minimum intervals to finish all tasks.

> [!TIP]
> Greedy + max-heap: always execute the most-frequent available task. After execution, put the task back into a cooldown queue with its next-available time. When no task is ready, CPU idles.

```python
import heapq
from collections import Counter, deque


def least_interval(tasks: list[str], n: int) -> int:
    """
    Max-heap (negated counts) + cooldown deque.
    O(t log t) where t = number of distinct tasks.
    """
    freq = Counter(tasks)
    # Max-heap: negate counts to turn min-heap into max-heap
    max_heap: list[int] = [-cnt for cnt in freq.values()]
    heapq.heapify(max_heap)

    # Cooldown queue: (next_available_time, negated_count)
    cooldown: deque[tuple[int, int]] = deque()
    time: int = 0

    while max_heap or cooldown:
        time += 1

        if max_heap:
            cnt = heapq.heappop(max_heap) + 1   # execute: decrement (less negative)
            if cnt < 0:
                cooldown.append((time + n, cnt))
        
        # Release tasks whose cooldown has expired
        if cooldown and cooldown[0][0] == time:
            _, cnt = cooldown.popleft()
            heapq.heappush(max_heap, cnt)

    return time


# Test
print(least_interval(["A","A","A","B","B","B"], 2))  # → 8
# Sequence: A B _ A B _ A B  (8 intervals)
```

## Sources

- Cormen, T. H., Leiserson, C. E., Rivest, R. L., & Stein, C. (2022). *Introduction to Algorithms* (4th ed.), Chapter 10 (Elementary Data Structures).
- Python documentation — `collections.deque`: https://docs.python.org/3/library/collections.html#collections.deque
- Python documentation — `queue.Queue`: https://docs.python.org/3/library/queue.html
- Python documentation — `heapq`: https://docs.python.org/3/library/heapq.html
- LeetCode problems: 232, 225, 622, 239, 347, 621.
- Source PDF: *8. Queues.pdf* (handwritten lecture notes, Data Structures and Algorithms course).

## Related

- [[06-linked-list]]
- [[07-stacks]]
- [[09-trees]]
- [[13-graphs]]
