---
title: "Strings"
created: 2026-05-19
updated: 2026-05-19
tags: [dsa, strings, pattern-matching, kmp, rabin-karp, sliding-window, palindrome, anagram]
aliases: ["String Algorithms", "05-strings"]
---

# Strings

[toc]

> **TL;DR:** A string is a sequence of characters drawn from a finite alphabet; in Python it is an immutable, Unicode code-point array that allocates a fresh object on every concatenation. The three foundational algorithms — naive O(nm) scan, Rabin-Karp rolling hash, and KMP prefix function — each trade preprocessing cost for a different worst-case guarantee, and knowing which to reach for (and why `"".join()` beats `+=` in a loop) separates correct code from fast code.

## Vocabulary

**String** — a finite, ordered sequence of characters drawn from an alphabet (ASCII, UTF-8, UTF-16, Unicode code points).

**Character set / encoding** — the mapping from characters to integers. ASCII covers 128 code points (7-bit). UTF-8 is a variable-width encoding of all Unicode code points; a Python `str` stores code points internally (CPython uses Latin-1, UCS-2, or UCS-4 depending on the highest code point in the string).

**Immutability** — a language-level guarantee that the bytes of an existing string object cannot change. Python `str` is immutable; every `s += x` allocates a new object and copies.

**Pattern matching** — given a text T of length n and a pattern P of length m, find all positions i where T[i:i+m] == P.

**Prefix function (failure function)** — for KMP, an array `lps[i]` = length of the longest proper prefix of `P[0..i]` that is also a suffix. Proper means strictly shorter than the full substring.

**Rolling hash** — a hash of a fixed-length window that can be updated in O(1) by subtracting the outgoing character's contribution and adding the incoming character's contribution.

**Suffix** — a contiguous tail substring of s; `s[i:]` for 0 ≤ i ≤ n.

**Palindrome** — a string that reads the same forwards and backwards; `s == s[::-1]`.

**Sliding window** — a two-pointer technique that maintains a contiguous subarray/substring satisfying some invariant, advancing left and right pointers to scan in O(n).

**Anagram** — a rearrangement of characters; two strings are anagrams iff they have the same character frequency multiset.

**Lexicographic rank** — the 0-based position of a string in the sorted list of all its permutations.

## Intuition

Think of a string as a read-only tape of integers: each cell holds a Unicode code point, and the tape is fixed in place after creation. Pattern matching is the problem of scanning a long tape (text) for a short template (pattern). The naive approach moves the template one cell at a time and re-checks every cell — O(nm) comparisons. The clever algorithms all share one insight: **when a mismatch occurs, we already have information about the pattern's internal structure that lets us skip ahead rather than backing up to i+1 and restarting.** KMP encodes that skip information in the prefix function. Rabin-Karp avoids most per-character comparisons by reducing each window to a single integer (a hash) and only doing a full character comparison when hashes match.

For palindromes, the key insight is symmetry: if you know the center of a palindrome, you can expand outward until characters disagree. Two-pointer and expand-around-center both exploit this O(n) center enumeration.

## Math foundations

The algorithms covered in this note are not just ad-hoc tricks — they rest on clean mathematical foundations: polynomial evaluation over a finite field, amortized pointer analysis, and string combinatorics. This section makes those foundations explicit so that complexity claims become derivable rather than memorized.

### Polynomial hashing — the math behind Rabin-Karp

A string s[0..n-1] over an alphabet of size σ can be read as the coefficient list of a degree-(n-1) polynomial evaluated at x = σ. The hash H(s) is just that polynomial evaluated at x = σ, reduced modulo a large prime p to keep the value bounded. Because polynomial evaluation is a linear map over the integers, every window of length m is a point on the same polynomial curve — and sliding the window is arithmetic, not character comparison.

```math
H(s[0..m-1]) = s[0] \cdot \sigma^{m-1} + s[1] \cdot \sigma^{m-2} + \cdots + s[m-1] \cdot \sigma^{0} \pmod{p}
```

Equivalently, H(s) = P(σ) where P(x) = Σ s[i] · x^(m-1-i) is a degree-(m-1) polynomial with character code-points as coefficients. The modular reduction mod p keeps every intermediate value in [0, p-1], making the entire computation fit in a machine word for typical p ≈ 10^9.

> [!IMPORTANT]
> p must be prime (not just large) so that the polynomial ring ℤ/pℤ has no zero-divisors. Using a power-of-2 modulus creates algebraic structure that an adversary can exploit to force collisions on many distinct strings.

### The rolling hash update is O(1)

Given the hash of the current window H(s[i..i+m-1]), computing the hash of the next window H(s[i+1..i+m]) requires only three arithmetic operations: subtract the leaving character's contribution, multiply by σ (shifting all remaining characters one degree higher), and add the new character. The key precomputation is σ^(m-1) mod p, done once in O(m) before the scan begins.

```math
H_{\text{new}} = \bigl( \sigma \cdot (H_{\text{old}} - s[i] \cdot \sigma^{m-1}) + s[i+m] \bigr) \pmod{p}
```

This is the heart of Rabin-Karp's O(n+m) expected time: the O(m) window check is only triggered on a hash match, which happens at most (n-m+1)/p ≈ 0 times on average for a good prime p. Across n-m sliding steps, the total work is O(n) for the rolling updates plus O(m) per verified match — negligible if matches are rare.

> [!TIP]
> In C implementations the intermediate value (H_old - s[i] * h_pow) can go negative before the mod. Always add p if the result is negative: `t_hash = ((t_hash - (text[i] * h) % q) % q + q) % q`. Python's `%` always returns non-negative, so this correction is only mandatory in C/C++.

### Collision probability and prime choice

Two distinct length-m strings s ≠ t can produce the same hash H(s) ≡ H(t) (mod p) — a spurious hit — only if their difference polynomial D(x) = P_s(x) - P_t(x) has σ as a root mod p. D is a nonzero polynomial of degree at most m-1, so it has at most m-1 roots in ℤ/pℤ. For a randomly chosen prime p from an interval [1, M], the probability of collision is at most (m-1) / M.

```math
\Pr[\text{collision}] \leq \frac{m-1}{p} \approx \frac{m}{p}
```

With p ≈ 10^9 and pattern length m ≤ 10^5, the collision probability per alignment is at most 10^(-4) — small but not negligible over n ≈ 10^6 alignments. The standard defense is **double hashing**: maintain two independent rolling hashes (σ₁, p₁) and (σ₂, p₂), and only confirm with a full O(m) character comparison when both hashes match. With two independent primes each around 10^9, the joint collision probability falls to m²/(p₁ · p₂) ≈ 10^(-8) per alignment, which is astronomically unlikely over any reasonable input.

### KMP failure function as a recurrence

The failure (prefix) function π[i] is defined as the length of the longest proper prefix of the pattern P[0..i] that is also a suffix of P[0..i]. "Proper" means strictly shorter than the whole substring — so π[i] < i+1 always. The recurrence builds π in one left-to-right pass: if the next character extends the current match, increment; otherwise fall back along the chain of already-computed prefix-suffix lengths until a match or the empty string.

```math
\pi[0] = 0
```

```math
\pi[i] = \begin{cases}
\pi[i-1] + 1 & \text{if } P[i] = P[\pi[i-1]] \\
\pi[\pi[\pi[i-1]-1]] + 1 & \text{(repeat fallback until match or 0)} \\
0 & \text{if no prefix matches}
\end{cases}
```

The amortized O(m) proof uses a potential function argument: let Φ = the current value of the length variable `len`. Each iteration of the inner while-loop decreases Φ by at least 1 (via `len = π[len-1]`), and `len` can only increase by 1 per outer loop step. Since `len` starts at 0 and can increase at most m times total, it can decrease at most m times total — so the while-loop body executes at most m times across the entire construction, giving O(m) amortized.

> [!NOTE]
> The fallback chain π[i], π[π[i]-1], π[π[π[i]-1]-1], … always terminates at 0 because the sequence of lengths is strictly decreasing and bounded below by 0. This is guaranteed by the "proper" constraint in the definition.

### Z-function — alternative to KMP

The Z-function of a string s is the array Z where Z[i] is the length of the longest substring starting at position i that matches a prefix of s. By convention Z[0] is undefined (or set to n). Like KMP, the Z-array enables linear-time pattern matching: append the pattern, a sentinel character not in the alphabet, then the text (s = P + '$' + T), compute Z, and any index i in the text portion with Z[i] = m is a match start.

```math
Z[i] = \max\{k \geq 0 : s[i..i+k-1] = s[0..k-1]\}
```

The linear-time Z-box construction maintains a rightmost Z-interval [l, r] — the interval [l, r] such that s[l..r] = s[0..r-l] and r is as large as possible among all intervals seen so far. For each new index i: if i ≤ r, initialize Z[i] = min(Z[i-l], r-i+1) using the mirror position i-l, then try to extend naively. Each naive extension step advances r, so r increases at most n times total across the whole loop — giving O(n) total work.

```math
Z[i] = \begin{cases}
\min(Z[i - l],\; r - i + 1) & \text{if } i \leq r \text{ (use Z-box mirror)} \\
0 & \text{if } i > r \text{ (start fresh)}
\end{cases}
\quad \text{then extend naively while } s[Z[i]] = s[i + Z[i]]
```


## How it works

Strings support a rich set of operations whose costs follow directly from the immutability and contiguous-memory layout. Every subsection below starts with the intuition before diving into the algorithm.

### Strings in memory

A Python `str` is stored as a contiguous array of Unicode code points. CPython selects the narrowest internal representation at construction time: Latin-1 (1 byte/char) if all code points fit in U+00FF, UCS-2 (2 bytes/char) if they fit in U+FFFF, and UCS-4 (4 bytes/char) otherwise. This means `len(s)` is always O(1) — the length is stored separately — and slicing `s[i:j]` is O(j-i) because it allocates and copies. Crucially, `s += x` is O(len(s) + len(x)) because it must allocate a new buffer and copy both strings into it.

```
Python str "hello" — memory layout (Latin-1 path, 1 byte per code point)
┌───────────────────────────────────────────────────────────────┐
│ PyUnicodeObject header: kind=1, length=5, hash=<cached>       │
├────┬────┬────┬────┬────┐                                       │
│ h  │ e  │ l  │ l  │ o  │  ← immutable byte array (code pts)  │
│104 │101 │108 │108 │111 │                                       │
└────┴────┴────┴────┴────┘                                       │
 [0]  [1]  [2]  [3]  [4]  indices, O(1) access
```

The implication for building strings in a loop: `s += c` inside a loop of n iterations gives O(n²) total work. Use a list and `"".join()` instead — it does one allocation and one copy.

```python
# O(n^2) — allocates a new string each iteration
def bad_join(parts: list[str]) -> str:
    s = ""
    for p in parts:
        s += p          # new allocation + copy every time
    return s

# O(n) — one allocation, one copy pass
def good_join(parts: list[str]) -> str:
    return "".join(parts)
```

> [!IMPORTANT]
> Python's CPython implementation has an optimization: if `s` has refcount 1 at the time of `+=`, it may resize in-place for short strings. Do NOT rely on this — it is an implementation detail that breaks under aliasing and disappears in PyPy/Jython.

### Naive matching

The naive algorithm slides the pattern P of length m over the text T of length n one position at a time. At each alignment i (0 ≤ i ≤ n-m), it checks all m characters. If any character mismatches, it advances i by 1 and restarts the inner loop from j=0. The algorithm is correct and simple, but its worst case — when the pattern almost-matches repeatedly — is O(nm). The classic degenerate input is text = "aaa...a" (n copies) and pattern = "aaa...ab" (m-1 'a's then 'b'): every alignment fails only at position m-1.

```python
def naive_search(text: str, pat: str) -> list[int]:
    """Return all start indices where pat occurs in text. O(nm) worst case."""
    n, m = len(text), len(pat)
    results: list[int] = []
    for i in range(n - m + 1):
        j = 0
        while j < m and text[i + j] == pat[j]:
            j += 1
        if j == m:
            results.append(i)
    return results
```

### Rabin-Karp rolling hash

Rabin-Karp avoids re-hashing every window from scratch. It treats each window of m characters as the coefficients of a polynomial evaluated at a chosen base `d` modulo a large prime `q`. When the window slides one character to the right, the old leading character is subtracted (multiplied out by `d^(m-1)`) and the new trailing character is added — this is an O(1) update. A hash match triggers a full O(m) character comparison to confirm (ruling out spurious hits). Expected complexity is O(n+m); worst case (all hashes collide) degrades to O(nm) but is extremely unlikely with a good prime.

```python
def rabin_karp(text: str, pat: str, d: int = 256, q: int = 101) -> list[int]:
    """
    Rabin-Karp pattern search.
    d: alphabet size / base
    q: a prime modulus to keep numbers bounded
    Returns all start indices of pat in text.
    """
    n, m = len(text), len(pat)
    if m > n:
        return []

    # h = d^(m-1) % q  — used to drop the leading character
    h: int = pow(d, m - 1, q)

    # Compute initial hash of pattern and first window of text
    p_hash: int = 0
    t_hash: int = 0
    for i in range(m):
        p_hash = (d * p_hash + ord(pat[i])) % q
        t_hash = (d * t_hash + ord(text[i])) % q

    results: list[int] = []
    for i in range(n - m + 1):
        if p_hash == t_hash:
            # Verify character by character to rule out spurious hit
            if text[i:i + m] == pat:
                results.append(i)
        if i < n - m:
            # Roll: subtract outgoing char, shift, add incoming char
            t_hash = (d * (t_hash - ord(text[i]) * h) + ord(text[i + m])) % q
            if t_hash < 0:
                t_hash += q
    return results
```

> [!TIP]
> For plagiarism detection or near-duplicate search, compute Rabin-Karp hashes for every k-gram across documents, then use a hash set to find matching k-grams in O(n) average time. This is exactly how academic plagiarism detectors work.

### KMP failure function

KMP (Knuth-Morris-Pratt) preprocesses the pattern to build the **prefix/failure function** array `lps` in O(m) time, then uses it to scan the text in O(n) time — O(n+m) total. The insight: when a mismatch occurs at pattern position j, we know that `text[i-j:i]` equals `pat[0:j]`. The `lps` table tells us the longest proper prefix of `pat[0:j]` that is also a suffix — meaning we can resume comparison at `pat[lps[j-1]]` rather than restarting from pat[0].

Building the `lps` table for `"ABABAC"`:

```
Pattern:  A  B  A  B  A  C
Index:    0  1  2  3  4  5

Step-by-step lps construction:
  i=0: lps[0] = 0  (base case: single char, no proper prefix)
  i=1: pat[1]='B' vs pat[lps[0]=0]='A' → mismatch; lps[1] = 0
  i=2: pat[2]='A' vs pat[0]='A' → match; lps[2] = 1
  i=3: pat[3]='B' vs pat[lps[2]=1]='B' → match; lps[3] = 2
  i=4: pat[4]='A' vs pat[lps[3]=2]='A' → match; lps[4] = 3
  i=5: pat[5]='C' vs pat[lps[4]=3]='B' → mismatch
        j = lps[2] = 1; pat[5]='C' vs pat[1]='B' → mismatch
        j = lps[0] = 0; pat[5]='C' vs pat[0]='A' → mismatch
        lps[5] = 0

lps array: [0, 0, 1, 2, 3, 0]

Meaning: if mismatch at position 4 ('A'), we know positions 0-2 ("ABA")
already matched → resume at j=3 instead of j=0.
```

```python
def build_lps(pat: str) -> list[int]:
    """Build the KMP failure function (longest proper prefix-suffix) in O(m)."""
    m = len(pat)
    lps: list[int] = [0] * m
    length = 0   # length of previous longest prefix-suffix
    i = 1
    while i < m:
        if pat[i] == pat[length]:
            length += 1
            lps[i] = length
            i += 1
        else:
            if length != 0:
                # Don't increment i; try a shorter prefix
                length = lps[length - 1]
            else:
                lps[i] = 0
                i += 1
    return lps


def kmp_search(text: str, pat: str) -> list[int]:
    """KMP pattern matching. O(n + m) time, O(m) space."""
    n, m = len(text), len(pat)
    if m == 0:
        return []
    lps = build_lps(pat)
    results: list[int] = []
    i = 0  # index into text
    j = 0  # index into pattern
    while i < n:
        if text[i] == pat[j]:
            i += 1
            j += 1
        if j == m:
            results.append(i - j)
            j = lps[j - 1]  # use lps to avoid re-scanning
        elif i < n and text[i] != pat[j]:
            if j != 0:
                j = lps[j - 1]
            else:
                i += 1
    return results
```

> [!WARNING]
> A common bug: after finding a full match (`j == m`), setting `j = 0` instead of `j = lps[m-1]` means overlapping matches are missed. Example: text="AABAA", pat="AA" should return [0, 3] but the naive reset returns only [0].

### Sliding window

The sliding window technique maintains two pointers `left` and `right` that bound a contiguous substring. The invariant is that the window always satisfies (or just violated) the target property. Advancing `right` expands the window; advancing `left` shrinks it. The key insight is that `left` never goes backward, so the total number of pointer movements is O(n). This gives O(n) time for problems like "longest substring without repeating characters" that would naively be O(n²).

```python
def length_of_longest_substring(s: str) -> int:
    """Longest substring without repeating characters. O(n) time, O(min(n,k)) space."""
    char_index: dict[str, int] = {}  # char -> most recent index seen
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in char_index and char_index[ch] >= left:
            # Move left past the previous occurrence to restore uniqueness
            left = char_index[ch] + 1
        char_index[ch] = right
        best = max(best, right - left + 1)
    return best
```

## Math

The polynomial rolling hash underlying Rabin-Karp:

```math
H(s[0..m-1]) = \left( \sum_{i=0}^{m-1} s[i] \cdot d^{m-1-i} \right) \mod q
```

where `d` is the base (e.g., 256 for byte-level) and `q` is a prime modulus. The rolling update when sliding one position right:

```math
H_{\text{new}} = \left( d \cdot \left( H_{\text{old}} - s[i] \cdot d^{m-1} \right) + s[i+m] \right) \mod q
```

This is O(1) per slide because `d^(m-1) mod q` is precomputed once.

KMP complexity recurrence: the text pointer `i` only advances (never retreats), and `j` can only decrease by the amount it has previously increased. Total increments of `i`: n. Total decrements of `j`: ≤ total increments of `j` ≤ n. So the scan loop runs in O(n), and the `lps` build runs in O(m) by the same argument — giving O(n+m) total.

```math
T_{\text{KMP}} = O(n + m), \quad S_{\text{KMP}} = O(m)
```

Naive matching worst-case:

```math
T_{\text{naive}} = O(n \cdot m)
```

Rabin-Karp expected (with negligible hash collision probability):

```math
T_{\text{RK}} = O(n + m) \text{ expected}, \quad O(nm) \text{ worst case}
```

## Real-world example

The Unix `grep` command is effectively a production pattern matcher. For fixed patterns, modern `grep` uses the Boyer-Moore-Horspool algorithm (a sibling of KMP with better average-case performance on English text). For the regex engine, patterns are compiled to a DFA/NFA. The following example reproduces the core of a simple log-search utility — scanning a large text for a fixed-string needle using KMP, then reporting line numbers.

```python
import sys
from typing import TextIO


def build_lps(pat: str) -> list[int]:
    m = len(pat)
    lps: list[int] = [0] * m
    length = 0
    i = 1
    while i < m:
        if pat[i] == pat[length]:
            length += 1
            lps[i] = length
            i += 1
        else:
            if length:
                length = lps[length - 1]
            else:
                lps[i] = 0
                i += 1
    return lps


def grep_kmp(stream: TextIO, pattern: str) -> None:
    """
    Stream line-by-line from a file and print lines containing pattern.
    Uses KMP so the pattern is compiled once and each line is scanned in O(len(line)).
    """
    lps = build_lps(pattern)
    m = len(pattern)
    for lineno, line in enumerate(stream, start=1):
        line = line.rstrip("\n")
        n = len(line)
        i = j = 0
        found = False
        while i < n:
            if line[i] == pattern[j]:
                i += 1
                j += 1
            if j == m:
                found = True
                break
            elif i < n and line[i] != pattern[j]:
                if j:
                    j = lps[j - 1]
                else:
                    i += 1
        if found:
            print(f"{lineno}: {line}")


if __name__ == "__main__":
    pattern = sys.argv[1]
    with open(sys.argv[2]) as fh:
        grep_kmp(fh, pattern)
```

> [!TIP]
> Precompile the `lps` table once outside the per-line loop. If you rebuild it inside the loop you pay O(m) per line, turning the overall complexity to O(n_lines * m + n_chars) instead of O(m + n_chars).

## In practice

**String concatenation cost in Python.** Every `s += piece` inside a loop is O(len(s)) because Python must allocate a new string object. Across n iterations this gives O(n²) total byte-copies. Use `parts: list[str] = []` then `"".join(parts)` — one allocation, one O(total) copy pass. This matters in practice: building a 1 MB string character by character via `+=` does ~500 GB of copy work; via `join` it does 1 MB.

**Unicode normalization traps.** The character "é" can be represented two ways in Unicode: as the precomposed code point U+00E9, or as "e" + combining acute accent (U+0065 U+0301). Both look identical on screen but `"é" == "é"` is `False` if they use different normal forms. Always normalize before comparing or hashing with `unicodedata.normalize("NFC", s)`. This bites multilingual applications constantly.

**Rabin-Karp modulus choice.** Using a non-prime modulus (e.g., a power of 2) dramatically increases hash collision probability because many character differences become invisible mod 2^k. Use a Mersenne prime (e.g., 2^31 - 1 = 2147483647) for better distribution. Also use double hashing (two independent (d, q) pairs) in adversarial settings where inputs may be chosen to cause collisions.

**KMP vs Boyer-Moore in practice.** KMP's O(n+m) is tight but has poor cache behavior — the `lps` array is accessed non-sequentially. Boyer-Moore scans right-to-left within each window and skips multiple characters on mismatches; for large alphabets and long patterns it is empirically 3-5x faster than KMP on English text. CPython's `str.find` uses a combination of Boyer-Moore and Horspool with a fast-path for short patterns.

> [!WARNING]
> CPython's `str.find` is extremely fast for ASCII text but can degrade on adversarially crafted UTF-8 inputs that force UCS-4 internal representation (any string containing a code point above U+FFFF). Profile before assuming `find` is O(n+m).

**`collections.Counter` for anagrams.** `Counter(s1) == Counter(s2)` is O(n) and correct for all Unicode. The frequency-array trick (`int[26]`) is O(n) only for pure lowercase-ASCII and silently gives wrong answers for any non-ASCII character.

## Pitfalls

- **Assuming `str` is mutable.** C/C++ programmers reach for in-place character replacement. In Python, `s[0] = 'x'` raises `TypeError`. Convert to `list(s)`, mutate, then `"".join(list)`.
- **O(n²) concat-in-loop.** `result = ""` then `result += s[i]` inside a loop. Always use `"".join()`. CPython's opportunistic in-place resize hides this in small tests but falls apart under aliasing or in other Python implementations.
- **Unicode vs bytes confusion.** `len("café")` is 4 (code points); `len("café".encode("utf-8"))` is 5 (bytes, because é encodes to 2 bytes). Slicing a bytes object at a UTF-8 boundary produces invalid UTF-8.
- **Failing to normalize Unicode before comparison.** `"é" == "é"` can be False; `unicodedata.normalize("NFC", a) == unicodedata.normalize("NFC", b)` is the correct check.
- **Forgetting the modulus sign in Rabin-Karp.** The rolling subtraction `(d * (h - ord(text[i]) * h_pow) + ord(text[i+m])) % q` can produce a negative intermediate value in Python (Python's `%` always returns non-negative, unlike C, so this is safe in Python but not in C). Always add q if the result is negative in C implementations.
- **KMP overlapping matches.** After a full match, resetting `j = 0` instead of `j = lps[m-1]` misses matches that overlap the current one.
- **Anagram check with set instead of Counter.** `set(s1) == set(s2)` checks unique characters, not frequencies. "aab" and "abb" would falsely compare equal.
- **Naive palindrome via string reversal in a loop.** `s == s[::-1]` is O(n) and idiomatic; doing it character by character with index arithmetic and getting the boundary wrong (off-by-one on odd vs even length) is a common source of bugs.

## Exercises

Each exercise below gives the problem statement, a hint, a full typed Python solution, and the complexity.

### Exercise 1 — Is Anagram

**Problem.** Given two strings `s` and `t`, return `True` if `t` is an anagram of `s` (same characters, same frequencies, possibly different order).

> [!NOTE]
> **Hint:** Two strings are anagrams iff their sorted forms are equal — but sorting is O(n log n). A frequency count brings it to O(n). `collections.Counter` handles Unicode correctly.

```python
from collections import Counter


def is_anagram(s: str, t: str) -> bool:
    """O(n) time, O(k) space where k = alphabet size."""
    return Counter(s) == Counter(t)


# Examples
assert is_anagram("anagram", "nagaram") is True
assert is_anagram("rat", "car") is False
assert is_anagram("a", "ab") is False
```

**Complexity.** O(n) time, O(k) space (k = number of distinct characters).

### Exercise 2 — Valid Palindrome (Alphanumeric, Case-Insensitive)

**Problem.** A phrase is a palindrome if, after removing all non-alphanumeric characters and converting to lowercase, it reads the same forwards and backwards. Given a string `s`, return `True` if it is a palindrome.

> [!NOTE]
> **Hint:** Two-pointer from both ends. Advance each pointer past non-alphanumeric characters before comparing. No extra string allocation needed.

```python
def is_palindrome(s: str) -> bool:
    """O(n) time, O(1) space — no cleaned string allocated."""
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True


assert is_palindrome("A man, a plan, a canal: Panama") is True
assert is_palindrome("race a car") is False
assert is_palindrome(" ") is True
```

**Complexity.** O(n) time, O(1) space.

### Exercise 3 — Longest Palindromic Substring (Expand Around Center)

**Problem.** Given a string `s`, return the longest palindromic substring. If there are ties, return any one.

> [!NOTE]
> **Hint:** For each of the 2n-1 centers (n odd-length centers between characters, n-1 even-length centers between adjacent characters), expand outward while characters match. Track the best start/end seen. Manacher's algorithm does this in O(n) by reusing previously computed palindrome radii, but the expand-around-center approach is O(n²) and far simpler to implement correctly.

```python
def longest_palindrome(s: str) -> str:
    """
    Expand-around-center. O(n^2) time, O(1) space.
    Manacher's algorithm achieves O(n) but is significantly more complex.
    """
    n = len(s)
    if n == 0:
        return ""

    best_start = 0
    best_len = 1

    def expand(left: int, right: int) -> None:
        nonlocal best_start, best_len
        while left >= 0 and right < n and s[left] == s[right]:
            left -= 1
            right += 1
        # After loop: s[left+1:right] is the palindrome
        length = right - left - 1
        if length > best_len:
            best_len = length
            best_start = left + 1

    for i in range(n):
        expand(i, i)      # odd-length palindromes
        expand(i, i + 1)  # even-length palindromes

    return s[best_start:best_start + best_len]


assert longest_palindrome("babad") in ("bab", "aba")
assert longest_palindrome("cbbd") == "bb"
assert longest_palindrome("a") == "a"
```

**Complexity.** O(n²) time, O(1) space. Manacher's algorithm reduces this to O(n) time, O(n) space by maintaining a rightmost palindrome boundary and exploiting mirror symmetry.

### Exercise 4 — Implement strStr (KMP)

**Problem.** Implement `strstr(haystack, needle)`: return the index of the first occurrence of `needle` in `haystack`, or -1 if not found. This is equivalent to `str.find`.

> [!NOTE]
> **Hint:** Use KMP for O(n+m) worst-case. The naive O(nm) solution will pass small inputs but fails under adversarial test cases like haystack="aaa...a" and needle="aaa...ab".

```python
def str_str(haystack: str, needle: str) -> int:
    """KMP-based strStr. O(n + m) time, O(m) space."""
    n, m = len(haystack), len(needle)
    if m == 0:
        return 0
    if m > n:
        return -1

    # Build lps (failure function)
    lps: list[int] = [0] * m
    length = 0
    i = 1
    while i < m:
        if needle[i] == needle[length]:
            length += 1
            lps[i] = length
            i += 1
        else:
            if length:
                length = lps[length - 1]
            else:
                lps[i] = 0
                i += 1

    i = j = 0
    while i < n:
        if haystack[i] == needle[j]:
            i += 1
            j += 1
        if j == m:
            return i - j
        elif i < n and haystack[i] != needle[j]:
            if j:
                j = lps[j - 1]
            else:
                i += 1
    return -1


assert str_str("sadbutsad", "sad") == 0
assert str_str("leetcode", "leeto") == -1
assert str_str("aabaabaab", "aab") == 0
```

**Complexity.** O(n+m) time, O(m) space.

### Exercise 5 — Longest Common Prefix

**Problem.** Given a list of strings, find the longest common prefix among all of them. Return an empty string if there is none.

> [!NOTE]
> **Hint:** Vertical scanning: compare characters column by column. Stop at the first column where any string differs or runs out. No sorting needed (unlike the trie approach, which is covered in the trees note).

```python
def longest_common_prefix(strs: list[str]) -> str:
    """Vertical scan. O(S) time where S = sum of all string lengths."""
    if not strs:
        return ""
    for col, ch in enumerate(strs[0]):
        for s in strs[1:]:
            if col >= len(s) or s[col] != ch:
                return strs[0][:col]
    return strs[0]


assert longest_common_prefix(["flower", "flow", "flight"]) == "fl"
assert longest_common_prefix(["dog", "racecar", "car"]) == ""
assert longest_common_prefix(["interview", "inter", "interesting"]) == "inter"
```

**Complexity.** O(S) time where S is the total character count of all strings. O(1) extra space.

### Exercise 6 — Group Anagrams

**Problem.** Given a list of strings, group the anagrams together. Return the groups in any order.

> [!NOTE]
> **Hint:** Two strings are anagrams iff their sorted forms are identical. Use the sorted form as a dict key. Sorting each string of length k takes O(k log k); with n strings of average length k the total is O(nk log k).

```python
from collections import defaultdict


def group_anagrams(strs: list[str]) -> list[list[str]]:
    """O(nk log k) time, O(nk) space. n = number of strings, k = max length."""
    groups: dict[str, list[str]] = defaultdict(list)
    for s in strs:
        key = "".join(sorted(s))
        groups[key].append(s)
    return list(groups.values())


result = group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
# Each inner list is an anagram group; order within groups may vary
assert sorted(sorted(g) for g in result) == [["ate", "eat", "tea"], ["bat"], ["nat", "tan"]]
```

**Complexity.** O(nk log k) time, O(nk) space.

### Exercise 7 — Leftmost Non-Repeating Character

**Problem.** Given a string, find the index of the first character that does not repeat. Return -1 if all characters repeat.

> [!NOTE]
> **Hint:** Two-pass: first build a frequency map (O(n)), then scan left-to-right for the first character with count 1 (O(n)). Using `collections.Counter` handles Unicode correctly.

```python
from collections import Counter


def first_non_repeating(s: str) -> int:
    """O(n) time, O(k) space (k = alphabet size)."""
    freq = Counter(s)
    for i, ch in enumerate(s):
        if freq[ch] == 1:
            return i
    return -1


assert first_non_repeating("geeksforgeeks") == 5   # 'f'
assert first_non_repeating("aabb") == -1
assert first_non_repeating("z") == 0
```

**Complexity.** O(n) time, O(k) space.

### Exercise 8 — Longest Substring Without Repeating Characters

**Problem.** Given a string `s`, find the length of the longest substring that contains no duplicate characters.

> [!NOTE]
> **Hint:** Classic sliding window. Maintain a dict mapping each character to its most recent index. When a duplicate is encountered inside the window (`char_index[ch] >= left`), advance `left` past the previous occurrence rather than shrinking one step at a time.

```python
def length_of_longest_substring_no_repeat(s: str) -> int:
    """Sliding window. O(n) time, O(min(n, k)) space."""
    seen: dict[str, int] = {}  # char -> last seen index
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in seen and seen[ch] >= left:
            left = seen[ch] + 1
        seen[ch] = right
        best = max(best, right - left + 1)
    return best


assert length_of_longest_substring_no_repeat("abcabcbb") == 3   # "abc"
assert length_of_longest_substring_no_repeat("bbbbb") == 1      # "b"
assert length_of_longest_substring_no_repeat("pwwkew") == 3     # "wke"
assert length_of_longest_substring_no_repeat("") == 0
```

**Complexity.** O(n) time, O(min(n, k)) space where k is the alphabet size.

## Sources

- Knuth, Morris, Pratt. "Fast Pattern Matching in Strings." SIAM Journal on Computing 6(2), 1977.
- Karp, Rabin. "Efficient Randomized Pattern-Matching Algorithms." IBM Journal of Research and Development 31(2), 1987.
- Cormen, Leiserson, Rivest, Stein. *Introduction to Algorithms* (CLRS), 4th ed. Chapter 32: String Matching.
- CPython source — `Objects/unicodeobject.c` and `Objects/bytes_methods.c` for `str.find` internals.
- Python documentation — `str` methods: https://docs.python.org/3/library/stdtypes.html#text-sequence-type-str
- Python documentation — `unicodedata.normalize`: https://docs.python.org/3/library/unicodedata.html
- Gusfield, Dan. *Algorithms on Strings, Trees and Sequences*. Cambridge University Press, 1997.
- Source PDF: "5. Strings" handwritten lecture notes (Data Structures and Algorithms course).

## Related

- [[04-hashing]] — rolling hash in Rabin-Karp directly applies hashing concepts; hash collision analysis lives there (forward reference: note not yet created)
- [[09-trees]] — trie is the canonical data structure for prefix lookup and replaces sorted-string approaches for longest common prefix at scale (forward reference: note not yet created)
- [[11-greedy-and-backtracking]] — pattern enumeration problems (permutations, combinations of strings) use backtracking; greedy appears in interval-merging on string positions (forward reference: note not yet created)
