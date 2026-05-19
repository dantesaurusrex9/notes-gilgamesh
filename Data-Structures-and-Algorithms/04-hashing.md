---
title: "Hashing"
created: 2026-05-19
updated: 2026-05-19
tags: [data-structures-and-algorithms, hashing, hash-table, collision-resolution, open-addressing, chaining, python-dict]
aliases: ["Hash Table", "Hash Map"]
---

# Hashing

[toc]

> **TL;DR:** A hash table maps arbitrary keys to fixed-size integer indices via a *hash function*, then stores values in a backing array at those indices — giving expected O(1) insert, delete, and search. Two fundamental strategies handle the inevitable collisions: *separate chaining* (each bucket owns a linked list of all colliding entries) and *open addressing* (collisions probe alternative slots within the same array). Python's `dict` is open-addressed with a compact layout that preserves insertion order since 3.6; understanding what breaks O(1) — degenerate hash functions, mutable keys, adversarial inputs — is as important as knowing why it works.

## Vocabulary

**Hash function** — a deterministic map h(k) from a key universe U to slot indices {0, 1, …, m-1}. Good hash functions are fast, uniform, and exhibit the avalanche effect.

**Hash table** — an array of m *buckets* backed by a hash function. Each bucket holds either a chained list head (chaining) or a key-value pair (open addressing).

**Bucket** — one slot in the backing array. Index i = h(k) % m.

**Load factor (α)** — the ratio n/m where n is the number of stored keys and m is the table capacity. Controls the collision probability and expected probe length.

**Collision** — two distinct keys k1 ≠ k2 with h(k1) = h(k2). Guaranteed by the pigeonhole principle when |U| > m.

**Separate chaining** — collision resolution where each bucket is the head of a singly-linked list. All keys that hash to bucket i form a chain at i.

**Open addressing** — collision resolution where all entries live in the array itself. A collision at slot i triggers a *probe sequence* to find an alternative slot.

**Linear probing** — open-addressing probe sequence: try i, i+1, i+2, … (mod m). Suffers primary clustering.

**Quadratic probing** — probe sequence: try i, i+1², i+2², … (mod m). Mitigates primary clustering; requires m prime and α < 0.5 for guaranteed termination.

**Double hashing** — probe sequence: try i, i+h2(k), i+2·h2(k), … (mod m) where h2 is a second independent hash function. Eliminates secondary clustering; m must be prime.

**Primary clustering** — long runs of occupied consecutive slots that form and grow under linear probing, inflating average probe length.

**Secondary clustering** — a weaker clustering effect in quadratic probing: different keys that collide at the same initial slot follow identical probe sequences.

**Rehashing** — allocating a new, larger table (typically 2× capacity) and re-inserting all keys when α exceeds a threshold (commonly 0.75 for chaining, 0.5 for open addressing).

**Universal hash family** — a set H of hash functions such that for any two distinct keys k1, k2, Pr[h(k1) = h(k2)] ≤ 1/m when h is drawn uniformly at random from H. Guarantees O(1) expected performance regardless of the input.

**Perfect hashing** — a collision-free hash function for a *static* known key set. Achieved in two levels (FKS scheme): O(n) space, O(1) worst-case lookup.

**Direct address table** — a special case where the key space is small enough to use keys directly as array indices (no hash function). arr[key] = 1 on insert, arr[key] = 0 on delete.

## Intuition

Think of m numbered pigeonholes on a wall. You want to file n documents. A good hash function acts like a fast, deterministic randomizer: it looks at each document's label and assigns it a pigeonhole number, spreading documents as evenly as possible. Two documents landing in the same hole is a *collision* — you either tape a sticky-note list to that hole (chaining) or look for the next empty hole (open addressing). The fundamental tension is between the array size m (memory) and the expected list/probe length (time): the load factor α = n/m is the single number that governs the trade-off.

Where hashing *fails*: when you need range queries ("all keys between 100 and 200"), sorted iteration, or prefix search. For those, reach for a balanced BST (AVL, red-black) or a trie — they give O(log n) ordered operations that hash tables cannot provide.

> [!NOTE]
> Hash tables are the second most-used data structure after arrays. They underlie Python `dict` and `set`, database indexes, compiler symbol tables, router forwarding tables, caches, and cryptographic integrity checks.

## Math foundations

The guarantees of hash tables — O(1) expected time, bounded collision probability, defence against adversarial keys — all rest on a handful of mathematical facts from modular arithmetic and probability. This section builds the formal scaffolding that underlies every claim in this note, so the intuition and probe-count formulas have a solid footing rather than appearing as magic numbers.

### Modular arithmetic

All hash functions ultimately reduce to arithmetic modulo the table size m. The two core congruence identities let you work with partial results without overflow: addition and multiplication distribute over mod, so you can reduce intermediate values at every step. These are the same identities behind polynomial rolling hash implementations.

```math
(a + b) \bmod m = \bigl((a \bmod m) + (b \bmod m)\bigr) \bmod m
```

```math
(a \cdot b) \bmod m = \bigl((a \bmod m) \cdot (b \bmod m)\bigr) \bmod m
```

The choice of m is not cosmetic. When m is a power of 2, h(k) = k mod m is equivalent to masking only the low-order bits — keys that are multiples of small factors cluster catastrophically. A **prime** m forces every remainder class to be equally reachable regardless of the arithmetic structure of the key set. In the polynomial rolling hash h(s) = Σ s_i · p^i mod m, choosing m prime (e.g. 10^9 + 7) and p coprime to m ensures the polynomial takes on all residue classes as s varies, giving maximum spread.

> [!IMPORTANT]
> m must be prime and far from any power of 2 for the division method to avoid patterned collisions. If m = 2^k, the hash is just a bitmask of the k least-significant bits — any set of keys that agree on those bits (e.g. all even integers) will collide 100% of the time.

### Probability primer for collisions

Assume **simple uniform hashing**: each key is equally likely to land in any of the m slots, independently. This is the idealization against which we measure real hash functions (universal hashing, below, achieves it without requiring any assumption about the input). Under this assumption, the probability that a single key hashes to slot j is exactly 1/m for any j.

Let n keys be inserted into a table of m slots using separate chaining. For a query key k, define an indicator variable X_i for each stored key k_i: X_i = 1 if h(k_i) = h(k), zero otherwise. The expected chain length at the bucket k hashes to is:

```math
E\!\left[\sum_{i=1}^{n} X_i\right] = \sum_{i=1}^{n} E[X_i] = \sum_{i=1}^{n} \frac{1}{m} = \frac{n}{m} = \alpha
```

This is linearity of expectation applied without any independence assumption between the X_i — the chain length in expectation is exactly the load factor α = n/m regardless of how keys are distributed, as long as each key individually satisfies the uniform assumption. A successful search costs 1 (the pointer dereference to the bucket) plus the expected chain length traversed before finding the key, which evaluates to 1 + α/2. An unsuccessful search traverses the whole chain: 1 + α.

> [!NOTE]
> Linearity of expectation requires no independence between events. This is the key move: even if collisions are correlated (as they are with a deterministic hash function on a structured input), E[sum] = sum of E[each] holds. Universal hashing is needed to ensure the per-key probability of 1/m holds — not to justify linearity.

### The birthday paradox

The birthday paradox quantifies how few keys are needed before a collision becomes more likely than not. With n keys hashed into m buckets, the probability that all n hash values are distinct (no collision yet) is:

```math
\Pr[\text{no collision after } n \text{ keys}] = \prod_{i=0}^{n-1} \left(1 - \frac{i}{m}\right)
\approx e^{-n(n-1)/(2m)}
```

using the approximation 1 - x ≈ e^(-x) for small x. The complementary collision probability is therefore:

```math
\Pr[\text{first collision by key } n] \approx 1 - e^{-n^2 / (2m)}
```

This exceeds 1/2 when n ≈ √(2m ln 2) ≈ 1.18 √m. The practical implication: **you need m >> n²** to keep collision probability low, or equivalently the table must be much larger than n when you care about worst-case behaviour. For a load factor of α = 0.75 (n = 0.75m), you are deep into the regime where collisions are routine — the table design must handle them gracefully rather than hoping to avoid them.

> [!WARNING]
> Cryptographic hash functions (SHA-256, BLAKE3) are designed to resist birthday attacks on a 256-bit output space (m = 2^256, so first collision at ~2^128 inputs). Algorithmic hash tables use 32–64 bit hashes, where birthday collisions occur at ~2^16–2^32 inputs. Never confuse the two security postures.

### Expected probe count math

Under the uniform hashing assumption for open addressing, the probability that any given slot in the probe sequence is occupied is at most α = n/m. Each probe is treated as an independent Bernoulli trial with success probability (1 - α) of finding an empty slot. The expected number of probes until the first empty slot (unsuccessful search) follows the geometric distribution:

```math
E[\text{probes, unsuccessful}] = \frac{1}{1 - \alpha}
```

For a successful search (finding a key that was inserted when the table had load α' < α), we average over all insertion moments. The key was inserted when the table had approximately α' keys; at that instant, the expected probe count was 1/(1 - α'). Averaging uniformly over α' ∈ [0, α]:

```math
E[\text{probes, successful}] = \frac{1}{\alpha} \int_0^{\alpha} \frac{1}{1 - \alpha'} \, d\alpha'
= \frac{1}{\alpha} \ln \frac{1}{1 - \alpha}
```

The cliff as α → 1 is severe in both formulas. At α = 0.9, unsuccessful probes cost 10×; at α = 0.99, they cost 100×. At α = 1 the table is full and an unsuccessful search never terminates.

```math
\lim_{\alpha \to 1^{-}} \frac{1}{1 - \alpha} = +\infty \qquad \lim_{\alpha \to 1^{-}} \frac{1}{\alpha} \ln \frac{1}{1 - \alpha} = +\infty
```

> [!IMPORTANT]
> These formulas assume each probe lands in a uniformly random slot — a condition that double hashing approximates well but linear probing violates due to primary clustering. For linear probing the correct Knuth formula for unsuccessful search is (1 + 1/(1-α)²)/2, which diverges faster. The practical consequence: keep α ≤ 0.7 for linear probing; α ≤ 0.5 for quadratic probing; α can reach 0.8–0.9 for double hashing or chaining without catastrophic degradation.

### Universal hashing

The uniform hashing assumption is fragile: an adversary who knows your hash function can construct a set of n keys that all map to the same bucket, reducing every operation to O(n). Universal hashing solves this by randomising over a *family* of hash functions rather than committing to one. A family H of functions from keys U to {0, …, m-1} is **universal** if for any two distinct keys x ≠ y:

```math
\Pr_{h \,\sim\, \mathcal{H}}\!\left[h(x) = h(y)\right] \leq \frac{1}{m}
```

where the probability is over the uniform random draw of h from H. This single inequality is enough to recover the same O(1) expected chain length derived above — the linearity-of-expectation argument goes through exactly as before, now with the 1/m collision probability guaranteed by the universal property rather than assumed from the key distribution. No matter what keys the adversary supplies, if h is drawn fresh and secretly at the start, the expected number of collisions involving any fixed key is at most n/m = α.

A concrete universal family: let p be a prime larger than |U|, pick a ≠ 0 and b uniformly at random from {0, …, p-1}, and define:

```math
h_{a,b}(k) = \bigl((a \cdot k + b) \bmod p\bigr) \bmod m
```

This is Carter and Wegman's (1979) construction. The family {h_{a,b} : a ∈ {1,…,p-1}, b ∈ {0,…,p-1}} is universal. Python's SipHash effectively draws from a large pseudo-random family by seeding with a per-process secret — it achieves the practical benefits of universal hashing without the theoretical overhead of picking from an explicit family at runtime.

> [!TIP]
> In practice you rarely implement a Carter–Wegman family directly. Instead, enable hash randomization (Python's default) or use a keyed MAC-based hash (SipHash, HighwayHash) with a fresh secret per deployment. These achieve the collision-resistance of universal hashing without the modular arithmetic overhead.

## How it works

A hash table is a backing array of m slots, plus a hash function h that maps each key k to a slot index h(k) ∈ {0, …, m-1}. The three operations — insert, search, delete — all start by computing h(k) and then applying the collision-resolution strategy to find the exact slot. The difference between chaining and open addressing is *where* the entry physically lives.

### Hash functions

A hash function compresses an arbitrarily large key space into a small integer range. Two classical methods appear in the PDF notes:

**Division method:** h(k) = k mod m. Simple and fast, but the choice of m matters enormously. If m is a power of 2, only the low-order bits of k influence the result — keys differing only in high bits map to the same slot. Choosing m to be a prime far from any power of 2 spreads bits more uniformly.

**Multiplication method:** h(k) = floor(m · (k · A mod 1)) where A ≈ (√5 − 1)/2 ≈ 0.618. This is Knuth's suggestion; it works for any m but is slightly slower.

**String hashing (polynomial rolling hash):** For a string s = s0 s1 … sL-1, compute the weighted sum of character ordinals with base p and modulus m. This is the hash used in the notes: h("abcd") = s[0]·p^0 + s[1]·p^1 + s[2]·p^2 + s[3]·p^3 mod m. A large prime p (e.g. 31 for lowercase ASCII) distributes characters well.

A good hash function must satisfy: it always produces the same value for the same input (deterministic); it maps keys *uniformly* across {0, …, m-1}; and it exhibits the *avalanche effect* — flipping one input bit flips roughly half the output bits, preventing adversarial clustering.

```python
# Division method — illustrative, not production
def hash_div(key: int, m: int) -> int:
    return key % m

# Bad hash: modulo a power of 2 loses high-order bits
def hash_bad(key: int) -> int:
    return key % 16  # ignores all bits above position 3

# Polynomial rolling hash for strings (p=31, m=large prime)
def hash_string(s: str, m: int = 1_000_000_007) -> int:
    p: int = 31
    result: int = 0
    p_pow: int = 1
    for ch in s:
        result = (result + (ord(ch) - ord('a') + 1) * p_pow) % m
        p_pow = (p_pow * p) % m
    return result
```

> [!IMPORTANT]
> m should be a prime number distant from any power of 2. If m = 2^k, the hash function degenerates to masking the k low-order bits — keys that differ only in high bits always collide.

### Separate chaining

In separate chaining the hash table is an array of m *linked-list heads*, initially all NULL. Each entry that hashes to bucket i is appended to the list at index i. The linked list holds (key, value) pairs; search walks the list comparing keys. Delete removes the node from its list. There is no upper bound on the number of entries per bucket — the list can grow arbitrarily (though doing so signals a bad hash function or extreme load).

**Figure: chaining memory layout.**

```
Backing array (m = 7 buckets)
 ┌─────┐
 │  0  │ → NULL
 ├─────┤
 │  1  │ → [key=71, val=A] → [key=15, val=B] → NULL
 ├─────┤
 │  2  │ → [key=56, val=C] → NULL
 ├─────┤
 │  3  │ → NULL
 ├─────┤
 │  4  │ → [key=58, val=D] → [key=23, val=E] → NULL
 ├─────┤
 │  5  │ → [key=19, val=F] → NULL
 ├─────┤
 │  6  │ → NULL
 └─────┘
  Buckets 1 and 4 have chains of length 2 (collisions).
  Insert 25: h(25) = 25 % 7 = 4 → prepend to bucket 4's chain.
```

> [!TIP]
> In the PDF's C++ implementation, `searchHighest` uses a sentinel value of -2 to mark empty slots. In Python, use `None` as the sentinel. When deleting from a chained table, you simply call `list.remove(key)` — no special tombstone is needed.

```python
from typing import Optional

class ChainingHashTable:
    """Separate-chaining hash table with typed slots."""

    def __init__(self, capacity: int = 8) -> None:
        self._m: int = capacity
        self._table: list[list[tuple[int, object]]] = [[] for _ in range(self._m)]
        self._n: int = 0

    def _h(self, key: int) -> int:
        return key % self._m

    def insert(self, key: int, value: object) -> None:
        bucket = self._h(key)
        for i, (k, _) in enumerate(self._table[bucket]):
            if k == key:
                self._table[bucket][i] = (key, value)  # update existing
                return
        self._table[bucket].append((key, value))
        self._n += 1
        if self._n / self._m > 0.75:
            self._rehash()

    def search(self, key: int) -> Optional[object]:
        bucket = self._h(key)
        for k, v in self._table[bucket]:
            if k == key:
                return v
        return None

    def delete(self, key: int) -> bool:
        bucket = self._h(key)
        for i, (k, _) in enumerate(self._table[bucket]):
            if k == key:
                self._table[bucket].pop(i)
                self._n -= 1
                return True
        return False

    def _rehash(self) -> None:
        old_table = self._table
        self._m *= 2
        self._table = [[] for _ in range(self._m)]
        self._n = 0
        for chain in old_table:
            for k, v in chain:
                self.insert(k, v)
```

### Open addressing

Open addressing stores every entry inside the backing array itself — no auxiliary linked lists. When a collision occurs at slot i, a *probe sequence* is followed to find an alternative empty slot. The array must therefore have capacity strictly greater than n at all times (α < 1 is required; α < 0.75 is typical).

**Figure: open-addressing memory layout (linear probing example).**

```
Insert keys: 50, 15, 51, 49, 16, 19  into m = 7

h(k) = k % 7

After all inserts:
 ┌───┬──────┐
 │ 0 │  49  │   ← h(49)=0; h(50)=1, probe 0+1=? no, 50→slot1
 ├───┼──────┤
 │ 1 │  50  │   ← h(50)=1 direct
 ├───┼──────┤
 │ 2 │  16  │   ← h(16)=2 direct
 ├───┼──────┤
 │ 3 │  51  │   ← h(51)=2 collision→probe 3
 ├───┼──────┤
 │ 4 │  19  │   ← h(19)=5 … probe wraps to 4? (depends on sequence)
 ├───┼──────┤
 │ 5 │  15  │   ← h(15)=1 collision→…→5
 ├───┼──────┤
 │ 6 │ EMPTY│
 └───┴──────┘
Probe path for key 51 (linear): slot 2 (occupied 16) → slot 3 (empty) ✓
```

**Linear probing** uses the probe sequence: slot(i) = (h(k) + i) % m. It is cache-friendly because probes are sequential in memory. The downside is *primary clustering* — occupied slots tend to coalesce into long runs, making probes longer over time.

**Quadratic probing** uses: slot(i) = (h(k) + i²) % m. This breaks up clusters of nearby slots. The catch is that quadratic probing only guarantees it finds a free slot if m is prime and α ≤ 0.5. Secondary clustering still occurs: two keys with the same initial slot follow the same probe sequence.

**Double hashing** uses: slot(i) = (h1(k) + i · h2(k)) % m where h2(k) = p − (k % p) for a prime p < m. Because each key has its own stride h2(k), different keys diverge after the first collision. This eliminates secondary clustering. m must be prime.

> [!WARNING]
> Delete in open addressing is non-trivial. You cannot simply clear the slot — doing so breaks probe sequences for keys that were inserted after the deleted key. Instead, mark deleted slots with a *tombstone* sentinel. Search skips tombstones but keeps probing; insert can reuse them. This is the source of the "dummy node" / tombstone pattern the PDF notes describe.

```python
_EMPTY = object()
_DELETED = object()   # tombstone

class OpenAddressHashTable:
    """Linear-probing open-address hash table."""

    def __init__(self, capacity: int = 8) -> None:
        self._m: int = capacity
        self._table: list[object] = [_EMPTY] * self._m
        self._keys: list[object] = [_EMPTY] * self._m
        self._n: int = 0

    def _h(self, key: int) -> int:
        return key % self._m

    def insert(self, key: int, value: object) -> None:
        if self._n / self._m >= 0.5:
            self._rehash()
        i: int = self._h(key)
        while self._keys[i] not in (_EMPTY, _DELETED):
            if self._keys[i] == key:
                self._table[i] = value
                return
            i = (i + 1) % self._m
        self._keys[i] = key
        self._table[i] = value
        self._n += 1

    def search(self, key: int) -> Optional[object]:
        i: int = self._h(key)
        while self._keys[i] is not _EMPTY:
            if self._keys[i] == key:
                return self._table[i]
            i = (i + 1) % self._m
        return None

    def delete(self, key: int) -> bool:
        i: int = self._h(key)
        while self._keys[i] is not _EMPTY:
            if self._keys[i] == key:
                self._keys[i] = _DELETED   # tombstone, not EMPTY
                self._table[i] = _DELETED
                self._n -= 1
                return True
            i = (i + 1) % self._m
        return False

    def _rehash(self) -> None:
        old_keys = self._keys[:]
        old_vals = self._table[:]
        self._m *= 2
        self._table = [_EMPTY] * self._m
        self._keys  = [_EMPTY] * self._m
        self._n = 0
        for k, v in zip(old_keys, old_vals):
            if k not in (_EMPTY, _DELETED):
                assert isinstance(k, int)
                self.insert(k, v)
```

### Resizing and rehashing

When the load factor α = n/m crosses the threshold (0.75 for chaining, 0.5 for open addressing with quadratic probing), the table must be resized. The standard strategy is to double m (to the next prime if open addressing), allocate a fresh table, and re-insert every live key-value pair. Because the hash function depends on m, every key maps to a different bucket in the new table — you cannot simply copy the old array.

Doubling gives amortized O(1) insertions: the total work across n inserts is O(n) even counting rehash events (the geometric series 1 + 2 + 4 + … + n sums to 2n). The per-insert worst case is O(n) during a rehash, but amortized cost stays constant.

> [!NOTE]
> The PDF notes use a threshold of α = 0.8 for the open-addressing demo and advise α ≤ 1 (load factor should be ≤ 1) for the table not to be full. In practice the real threshold is lower: 0.5 for quadratic probing (mathematical requirement), 0.7–0.75 for linear probing (empirical quality), 0.75 for chaining (Python `dict` default).

## Math

The expected-cost analysis below assumes *simple uniform hashing*: each key is equally likely to hash to any of the m slots, independently of other keys. This is an idealization — real hash functions approximate it, and universal hash families achieve it in expectation.

**Expected probe count under chaining (successful search):**

```math
E[\text{probes}] \approx 1 + \frac{\alpha}{2}
```

**Expected probe count under chaining (unsuccessful search / insert):**

```math
E[\text{probes}] \approx 1 + \alpha
```

**Expected probe count under open addressing (unsuccessful search), uniform hashing:**

```math
E[\text{probes}] \leq \frac{1}{1 - \alpha}
```

**Expected probe count under open addressing (successful search):**

```math
E[\text{probes}] \leq \frac{1}{\alpha} \ln \frac{1}{1 - \alpha}
```

**Comparison at representative load factors:**

| α | Chaining (unsucc.) | Open addr. (unsucc.) |
| :---: | :---: | :---: |
| 0.50 | 1.5 | 2.0 |
| 0.75 | 1.75 | 4.0 |
| 0.90 | 1.9 | 10.0 |
| 0.95 | 1.95 | 20.0 |

Open addressing degrades dramatically past α = 0.75 — this is why Python `dict` rehashes at 2/3 occupancy and why quadratic probing mandates α < 0.5.

**String polynomial hash:**

```math
h(s) = \left(\sum_{i=0}^{L-1} s_i \cdot p^{i}\right) \bmod m
```

where p is a prime base (31 for lowercase ASCII, 131 for arbitrary bytes) and m is a large prime (e.g. 10^9 + 7).

**Double hashing probe formula:**

```math
\text{slot}(i) = \bigl(h_1(k) + i \cdot h_2(k)\bigr) \bmod m, \quad h_2(k) = p - (k \bmod p)
```

where p < m is prime. This guarantees all m slots are visited before repeating (the stride h2(k) and m are coprime when m is prime).

> [!IMPORTANT]
> The 1/(1−α) formula for open addressing assumes each probe is an *independent* uniform random sample. Real linear probing violates this assumption — actual probe costs under linear probing with primary clustering are higher. The correct analysis for linear probing is due to Knuth and yields roughly (1 + 1/(1−α)²) / 2 expected probes for an unsuccessful search.

## Real-world example

A common pattern in web services is *request deduplication*: an API gateway that receives retried POST requests must ensure idempotency — the same logical request should not be processed twice. The gateway stores a hash map from request-id to response, with a short TTL. Here is a Python implementation using `dict` with an LRU eviction policy layered on top:

```python
from __future__ import annotations

import time
from collections import OrderedDict
from typing import Any

class RequestDedupeCache:
    """
    O(1) idempotency cache backed by Python dict (open-addressed, insertion-ordered).

    Uses an OrderedDict so LRU eviction is O(1): the oldest entry is always at
    the front. TTL is enforced lazily on read to avoid background thread complexity.
    """

    def __init__(self, max_size: int = 10_000, ttl_seconds: float = 300.0) -> None:
        self._store: OrderedDict[str, tuple[Any, float]] = OrderedDict()
        self._max_size: int = max_size
        self._ttl: float = ttl_seconds

    def get(self, request_id: str) -> tuple[bool, Any]:
        """Return (found, cached_response). Evicts stale entries on hit."""
        if request_id not in self._store:
            return False, None
        response, ts = self._store[request_id]
        if time.monotonic() - ts > self._ttl:
            del self._store[request_id]
            return False, None
        # Move to end (most-recently-used position)
        self._store.move_to_end(request_id)
        return True, response

    def put(self, request_id: str, response: Any) -> None:
        """Cache a response. Evicts LRU entry if over capacity."""
        if request_id in self._store:
            self._store.move_to_end(request_id)
        self._store[request_id] = (response, time.monotonic())
        if len(self._store) > self._max_size:
            self._store.popitem(last=False)  # evict oldest (LRU)


# Usage
cache = RequestDedupeCache(max_size=1000, ttl_seconds=60.0)

def handle_request(request_id: str, payload: dict[str, Any]) -> dict[str, Any]:
    found, resp = cache.get(request_id)
    if found:
        return resp  # type: ignore[return-value]
    result: dict[str, Any] = {"status": "processed", "data": payload}
    cache.put(request_id, result)
    return result
```

> [!TIP]
> `OrderedDict.move_to_end` and `popitem(last=False)` are O(1) because CPython implements `OrderedDict` as a doubly-linked list threaded through the dict's internal hash table nodes. This is more efficient than sorting or searching the dict on every eviction.

## In practice

### Python `dict` internals

CPython's `dict` is an open-addressed hash table using a *compact* layout introduced in Python 3.6. Before 3.6, the backing array held `(hash, key, value)` triples — deletion of a key left a tombstone hole, and iteration order was unspecified. The compact layout separates concerns: a small *indices* array (of integers) maps slot → position in a dense *entries* array of `(hash, key, value)` triples. Entries are always appended in insertion order; the indices array is rehashed on resize. This gives:

- Insertion-order iteration (guaranteed since Python 3.7).
- ~20–25% smaller memory footprint than the old layout because the indices array is smaller per slot.
- The same O(1) amortized operations with a load-factor threshold of 2/3.

Python's probing is not pure linear — it uses a pseudo-random perturbation: `slot = (5*slot + 1 + perturb) % m; perturb >>= 5`. This avoids the primary clustering of naive linear probing while remaining CPU-cache-friendly.

### Hashable types and the mutable-key footgun

Python's data model requires hashable objects to implement `__hash__` and `__eq__` with the invariant: objects that compare equal must have the same hash. Mutable containers (`list`, `dict`, `set`) are not hashable because mutating them after insertion would change their hash — the object would be filed under a different bucket than where it lives, making it permanently unfindable.

> [!CAUTION]
> Using a mutable object as a dict key silently corrupts the table. Python forbids `list` and `dict` keys at the type level, but a custom class with a mutable `__hash__` can slip through. If you mutate an instance after using it as a key, the key becomes permanently unfindable — no error, no warning, just a silent bug.

### Hash DoS and SipHash

Prior to Python 3.3, string hashing was deterministic across processes. An attacker who knew the hash function could craft many strings that all hashed to the same bucket, turning O(1) dict operations into O(n) and creating a denial-of-service vector in any web server that fed user input into a `dict` (e.g. HTTP query-string parsing). Python 3.3 introduced *hash randomization*: a per-process random seed is mixed into string, bytes, and datetime hashes. Since Python 3.6 this uses SipHash-1-3, a fast pseudo-random-function designed for hash tables. The seed is unpredictable across process restarts (`PYTHONHASHSEED=random` by default). You can set `PYTHONHASHSEED=0` to disable randomization for reproducibility in tests, but never in production services.

### When not to use a hash table

| Need | Use instead |
| :--- | :--- |
| Range query: all keys in [lo, hi] | Balanced BST (AVL, red-black) — O(log n) |
| Sorted iteration | Same; or sorted array with binary search |
| Prefix search ("all keys starting with 'geo'") | Trie — O(L) per query |
| Nearest-neighbor by numeric value | B-tree / skip list |
| Persistent, disk-backed map | B-tree (database index) |

## Pitfalls

- **"O(1) is guaranteed."** — Worst-case cost is O(n) when all keys collide into one bucket (either a degenerate hash function or a malicious input without hash randomization). O(1) is *expected* under uniform hashing, or worst-case with universal hashing.

- **"I can use a list as a dict key."** — Python will raise `TypeError: unhashable type: 'list'` immediately. The deeper pitfall is a *custom object* with a mutable `__hash__` — Python won't prevent this, and mutating the object after using it as a key silently loses the entry.

- **"Deleting from an open-addressed table is just clearing the slot."** — Clearing without a tombstone breaks probe chains. Any key that probed through the cleared slot becomes unreachable. Always use a tombstone sentinel.

- **"Quadratic probing always finds a free slot."** — It only guarantees full coverage if m is prime and α < 0.5. With a composite m or α ≥ 0.5, quadratic probing can cycle through a subset of slots and fail to find a free one even when free slots exist.

- **"My hash is fine because it passes all my tests."** — Deterministic hash functions are vulnerable to adversarial inputs. Web-facing servers must use randomized hashing (SipHash) or universal hashing.

- **"Rehashing is just copying the array."** — You must re-compute h(k) for every key because the new m changes every bucket index. Copying raw slots produces a corrupt table.

- **"A hash table preserves insertion order in Python."** — True for `dict` since Python 3.7, but `set` makes no ordering guarantee (it uses a different internal layout). Do not rely on `set` iteration order.

- **"A higher load factor is always better (saves memory)."** — True for chaining up to a point, catastrophic for open addressing. At α = 0.9, expected unsuccessful probes under open addressing is 10×; at α = 0.95, it is 20×. The memory saving is not worth the performance cliff.

## Exercises

The exercises below cover the standard hash-table problem patterns. Each includes the problem statement, a hint pointing to the key insight, a typed Python solution, and complexity. They draw from the PDF's C++ examples and canonical LeetCode problems.

---

### Exercise 1: Two Sum

**Problem:** Given a list of integers `nums` and a target integer `target`, return the indices [i, j] such that nums[i] + nums[j] == target. Exactly one solution exists.

**Hint:** For each element x, the complement is `target - x`. Store elements seen so far in a dict mapping value → index. Check if the complement is already present before inserting.

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    seen: dict[int, int] = {}   # value -> index
    for i, x in enumerate(nums):
        complement = target - x
        if complement in seen:
            return [seen[complement], i]
        seen[x] = i
    return []   # guaranteed to find a solution by problem statement

# Example
assert two_sum([2, 7, 11, 15], 9) == [0, 1]
assert two_sum([3, 2, 4], 6) == [1, 2]
```

Time O(n), Space O(n).

---

### Exercise 2: Group Anagrams

**Problem:** Given a list of strings, group all anagrams together. Return a list of groups.

**Hint:** Two strings are anagrams iff their sorted forms are equal. Use the sorted string as the dict key.

```python
from collections import defaultdict

def group_anagrams(strs: list[str]) -> list[list[str]]:
    groups: dict[str, list[str]] = defaultdict(list)
    for s in strs:
        key = "".join(sorted(s))
        groups[key].append(s)
    return list(groups.values())

# Example
result = group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
# [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']] (order within groups may vary)
assert len(result) == 3
```

Time O(n · k log k) where k = max string length. Space O(n·k).

---

### Exercise 3: First Non-Repeating Character

**Problem:** Given a string, find the index of the first character that appears exactly once. Return -1 if none exists.

**Hint:** Two passes: first pass builds a frequency dict; second pass finds the first character with count 1.

```python
def first_non_repeating(s: str) -> int:
    freq: dict[str, int] = {}
    for ch in s:
        freq[ch] = freq.get(ch, 0) + 1
    for i, ch in enumerate(s):
        if freq[ch] == 1:
            return i
    return -1

# Examples
assert first_non_repeating("leetcode") == 0   # 'l'
assert first_non_repeating("aabb") == -1
assert first_non_repeating("loveleetcode") == 2  # 'v'
```

Time O(n), Space O(1) (at most 26 keys for lowercase ASCII).

---

### Exercise 4: Longest Substring Without Repeating Characters

**Problem:** Given a string s, find the length of the longest substring without repeating characters.

**Hint:** Sliding window with a hash map storing the most recent index of each character. When a duplicate is detected, advance the left pointer past the previous occurrence.

```python
def length_of_longest_substring(s: str) -> int:
    last_seen: dict[str, int] = {}
    left: int = 0
    best: int = 0
    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1
        last_seen[ch] = right
        best = max(best, right - left + 1)
    return best

# Examples
assert length_of_longest_substring("abcabcbb") == 3   # "abc"
assert length_of_longest_substring("bbbbb") == 1
assert length_of_longest_substring("pwwkew") == 3     # "wke"
```

Time O(n), Space O(min(n, alphabet_size)).

---

### Exercise 5: Subarray Sum Equals K (prefix sum + hash)

**Problem:** Given an integer array `nums` and integer `k`, return the count of contiguous subarrays whose sum equals k.

**Hint:** The sum of subarray [i+1 … j] equals prefix[j] − prefix[i]. If prefix[j] − k = prefix[i] for some i < j, that subarray has sum k. Store prefix sum frequencies in a dict; initialize with {0: 1} to handle subarrays starting from index 0.

```python
def subarray_sum(nums: list[int], k: int) -> int:
    prefix_count: dict[int, int] = {0: 1}
    current_sum: int = 0
    count: int = 0
    for x in nums:
        current_sum += x
        # How many previous prefix sums equal current_sum - k?
        count += prefix_count.get(current_sum - k, 0)
        prefix_count[current_sum] = prefix_count.get(current_sum, 0) + 1
    return count

# Examples
assert subarray_sum([1, 1, 1], 2) == 2
assert subarray_sum([1, 2, 3], 3) == 2
assert subarray_sum([-1, -1, 1], 0) == 1
```

Time O(n), Space O(n). The naive two-loop approach is O(n²).

---

### Exercise 6: Top-K Frequent Elements

**Problem:** Given an integer array `nums` and integer `k`, return the k most frequent elements.

**Hint:** Build a frequency dict, then use bucket sort (index = frequency) for O(n) extraction. The frequency of any element is at most n, so bucket indices fit in an array of size n+1.

```python
def top_k_frequent(nums: list[int], k: int) -> list[int]:
    freq: dict[int, int] = {}
    for x in nums:
        freq[x] = freq.get(x, 0) + 1

    # Bucket sort: bucket[f] = list of elements with frequency f
    buckets: list[list[int]] = [[] for _ in range(len(nums) + 1)]
    for val, f in freq.items():
        buckets[f].append(val)

    result: list[int] = []
    for f in range(len(buckets) - 1, 0, -1):
        result.extend(buckets[f])
        if len(result) >= k:
            break
    return result[:k]

# Examples
assert set(top_k_frequent([1, 1, 1, 2, 2, 3], 2)) == {1, 2}
assert top_k_frequent([1], 1) == [1]
```

Time O(n), Space O(n). An O(n log k) heap alternative: `heapq.nlargest(k, freq.items(), key=lambda p: p[1])`.

---

### Exercise 7: Isomorphic Strings

**Problem:** Given strings s and t, determine if they are isomorphic. Two strings are isomorphic if each character in s can be replaced consistently (one-to-one) to get t.

**Hint:** Maintain two maps: s_char → t_char and t_char → s_char. On each position, both mappings must be consistent.

```python
def is_isomorphic(s: str, t: str) -> bool:
    s_to_t: dict[str, str] = {}
    t_to_s: dict[str, str] = {}
    for cs, ct in zip(s, t):
        if cs in s_to_t:
            if s_to_t[cs] != ct:
                return False
        else:
            if ct in t_to_s and t_to_s[ct] != cs:
                return False
            s_to_t[cs] = ct
            t_to_s[ct] = cs
    return True

# Examples
assert is_isomorphic("egg", "add") is True
assert is_isomorphic("foo", "bar") is False
assert is_isomorphic("paper", "title") is True
```

Time O(n), Space O(alphabet_size).

---

### Exercise 8: Design a Hash Map From Scratch

**Problem:** Implement `MyHashMap` with `put(key, val)`, `get(key)` (return -1 if missing), and `remove(key)` without using any built-in hash table libraries.

**Hint:** Use separate chaining with a fixed array of m = 1009 buckets (a prime). Each bucket is a list of `[key, value]` pairs.

```python
class MyHashMap:
    """Separate-chaining hash map. Keys and values are non-negative integers."""

    _M: int = 1009   # prime bucket count

    def __init__(self) -> None:
        self._table: list[list[list[int]]] = [[] for _ in range(self._M)]

    def _h(self, key: int) -> int:
        return key % self._M

    def put(self, key: int, val: int) -> None:
        bucket = self._table[self._h(key)]
        for pair in bucket:
            if pair[0] == key:
                pair[1] = val
                return
        bucket.append([key, val])

    def get(self, key: int) -> int:
        for pair in self._table[self._h(key)]:
            if pair[0] == key:
                return pair[1]
        return -1

    def remove(self, key: int) -> None:
        bucket = self._table[self._h(key)]
        for i, pair in enumerate(bucket):
            if pair[0] == key:
                bucket.pop(i)
                return

# Smoke test
hm = MyHashMap()
hm.put(1, 10); hm.put(2, 20)
assert hm.get(1) == 10
assert hm.get(3) == -1
hm.put(2, 99)
assert hm.get(2) == 99
hm.remove(2)
assert hm.get(2) == -1
```

Time O(n/m) amortized per operation (O(1) with a good hash and low load). Space O(n + m).

## Sources

- Cormen, T. H. et al. *Introduction to Algorithms*, 4th ed. MIT Press, 2022. Chapters 11 (Hash Tables) and 17 (Amortized Analysis).
- Knuth, D. E. *The Art of Computer Programming*, Vol. 3: Sorting and Searching. Section 6.4 (Hashing).
- Python documentation — `dict` implementation notes: https://docs.python.org/3/faq/design.html#how-are-dictionaries-implemented-in-cpython
- Raymond Hettinger, "Modern Dictionaries at PyCon 2017" (compact dict layout explanation): https://www.youtube.com/watch?v=p33CVV29OG8
- Fredman, M., Komlós, J., Szemerédi, E. (1984). "Storing a Sparse Table with O(1) Worst Case Access Time." *Journal of the ACM* 31(3). (FKS perfect hashing.)
- Bernstein, D. J. (2012). SipHash: https://131002.net/siphash/
- Source PDF: *4. Hashing.pdf* — handwritten lecture notes, Data Structures and Algorithms course.

## Related

- [[02-arrays-and-searching]]
- [[05-strings]]
- [[06-linked-list]]
