---
title: "Math for Technical Interviews"
created: 2026-06-10
updated: 2026-06-10
tags: []
aliases: []
---

# Math for Technical Interviews

[toc]

> **TL;DR:** Most software engineering interviews need practical high-school math, not advanced university math. The useful pieces are number bases, logarithms, factorials/permutations, subsets, sequence sums, modulo, and enough divisibility reasoning to analyze simple algorithms.

## Vocabulary

**Base**

```math
base = b
```

The number of unique digits used in a numeral system. Decimal uses base 10; binary uses base 2.

**Exponent**

```math
b^k
```

Repeated multiplication by a base. For example, 2 to the power of 4 equals 16.

**Logarithm**

```math
\log_b(n) = k \iff b^k = n
```

The inverse of exponentiation. It asks: what power must I raise the base to in order to get n?

**Permutation**

```math
n!
```

An ordering of distinct items. Factorial counts how many ways n distinct items can be arranged.

**Subset**

```math
2^n
```

A selection of elements from a set. Each element is either included or excluded, so n elements create 2 to the n possible subsets.

**Arithmetic sequence**

```math
a,\ a+d,\ a+2d,\ a+3d,\ldots
```

A sequence where consecutive terms differ by a constant amount.

**Geometric sequence**

```math
a,\ ar,\ ar^2,\ ar^3,\ldots
```

A sequence where consecutive terms have a constant ratio.

**Modulo**

```math
a \bmod m
```

The remainder after dividing a by m. In Python and most interview code, modulo is written with `%`.

## What Math Usually Matters

For most software engineering interviews, the useful math is practical. You need enough to reason about loops, splitting problems in half, counting choices, and checking divisibility.

You usually do not need advanced calculus, abstract algebra, or obscure math contest tricks. If a problem depends on one rare formula, that is usually less representative of normal interview expectations than arrays, strings, hash maps, trees, graphs, recursion, and dynamic programming.

> [!NOTE]
> Some LeetCode problems use special math tricks, but a giant question bank is not the same thing as a normal interview signal. Prioritize reusable reasoning over memorizing rare facts.

## Number Bases

A number base tells you how much each digit position is worth. In base 10, each position is a power of 10.

```math
352 = 3 \cdot 10^2 + 5 \cdot 10^1 + 2 \cdot 10^0
```

Binary is base 2. Computers use binary because digital circuits naturally represent two states, often thought of as off/on or false/true.

```math
1011_2 = 1 \cdot 2^3 + 0 \cdot 2^2 + 1 \cdot 2^1 + 1 \cdot 2^0 = 11_{10}
```

This small Python example shows the same idea directly.

```python
def binary_to_decimal(bits: str) -> int:
    total = 0
    for bit in bits:
        total = total * 2 + int(bit)
    return total


assert binary_to_decimal("1011") == 11
```

## Logarithms

A logarithm answers the question: how many times can this problem shrink by a base before it reaches 1?

For base 2:

```math
2^3 = 8
```

```math
\log_2(8) = 3
```

This is why logarithms appear in binary search, balanced trees, heaps, and divide-and-conquer. Those algorithms repeatedly cut the problem in half.

```python
def divisions_by_two_until_one(n: int) -> int:
    count = 0
    while n > 1:
        n //= 2
        count += 1
    return count


assert divisions_by_two_until_one(8) == 3
assert divisions_by_two_until_one(16) == 4
```

When you see an algorithm repeatedly halve the remaining work, think O(log n).

## Permutations and Factorials

A permutation is an arrangement. If you have three letters `a`, `b`, and `c`, there are six ways to arrange them.

```math
3! = 3 \cdot 2 \cdot 1 = 6
```

For n distinct items, the first position has n choices, the second has n minus 1 choices, and so on.

```math
n! = n \cdot (n - 1) \cdot (n - 2) \cdots 1
```

This matters in interviews because generating all permutations grows extremely fast.

```python
def factorial(n: int) -> int:
    result = 1
    for value in range(2, n + 1):
        result *= value
    return result


assert factorial(5) == 120
```

If a brute-force solution generates every permutation, be suspicious. Factorial growth becomes unusable quickly.

## Subsets and Powersets

For each element in a set, you make one yes/no decision: include it or exclude it. That gives two choices per element.

```math
number\ of\ subsets = 2^n
```

For three items, there are 8 subsets.

```python
def subsets(items: list[int]) -> list[list[int]]:
    result: list[list[int]] = [[]]

    for item in items:
        result += [current + [item] for current in result]

    return result


assert subsets([1, 2, 3]) == [
    [],
    [1],
    [2],
    [1, 2],
    [3],
    [1, 3],
    [2, 3],
    [1, 2, 3],
]
```

When a problem asks for every subset, O(2^n) is often unavoidable because the output itself has that many results.

## Arithmetic Sequences

An arithmetic sequence adds the same amount each time.

```math
1,\ 2,\ 3,\ 4,\ 5
```

The sum of an arithmetic sequence is:

```math
sum = \frac{(first + last) \cdot count}{2}
```

This is useful for nested loop analysis.

```python
def triangular_work(n: int) -> int:
    count = 0
    for i in range(n):
        for _ in range(i + 1):
            count += 1
    return count


assert triangular_work(5) == 15
```

The total work is:

```math
1 + 2 + 3 + \cdots + n = \frac{n(n + 1)}{2}
```

For Big-O, this is O(n squared).

## Geometric Sequences

A geometric sequence multiplies by the same ratio each time.

```math
1,\ 2,\ 4,\ 8,\ 16
```

The sum of the first n terms is:

```math
sum = first \cdot \frac{1 - ratio^n}{1 - ratio}
```

In interviews, this often appears in trees. A perfect binary tree has 1 node at level 0, 2 nodes at level 1, 4 nodes at level 2, and so on.

```python
def perfect_binary_tree_nodes(height: int) -> int:
    total = 0
    nodes_at_level = 1

    for _ in range(height + 1):
        total += nodes_at_level
        nodes_at_level *= 2

    return total


assert perfect_binary_tree_nodes(0) == 1
assert perfect_binary_tree_nodes(2) == 7
assert perfect_binary_tree_nodes(3) == 15
```

This is why a complete binary tree of height h has about 2 to the h nodes, and why height is about log n.

## Modular Arithmetic

Modulo gives the remainder after division. It is "wraparound" arithmetic, like clocks.

```math
15 \bmod 12 = 3
```

In Python:

```python
assert 15 % 12 == 3
assert 32 % 12 == 8
```

Modulo is useful when a problem involves divisibility, cyclic behavior, remainders, clocks, array rotation, hashing, or "return the answer modulo some number."

One useful property is distributivity over addition:

```math
(a + b) \bmod c = ((a \bmod c) + (b \bmod c)) \bmod c
```

That property lets you keep numbers small while accumulating a large sum.

```python
def sum_mod(values: list[int], mod: int) -> int:
    total = 0
    for value in values:
        total = (total + value) % mod
    return total


assert sum_mod([13, 2], 12) == 3
```

## Primality Check

A prime number has exactly two positive divisors: 1 and itself. The number 1 is not prime.

To check if n is prime, you only need to test divisors up to the square root of n. If n has a factor larger than its square root, the matching paired factor must be smaller than the square root.

```python
def is_prime(n: int) -> bool:
    if n < 2:
        return False

    divisor = 2
    while divisor * divisor <= n:
        if n % divisor == 0:
            return False
        divisor += 1

    return True


assert not is_prime(1)
assert not is_prime(15)
assert is_prime(17)
```

The modulo check `n % divisor == 0` means divisor divides n evenly.

## Interview Pattern Map

Use this as a quick recognition table while practicing.

| Concept | Interview signal | Usual complexity clue |
| --- | --- | --- |
| Logarithm | repeatedly halve the search space | O(log n) |
| Factorial | generate every ordering | O(n!) |
| Subsets | include/exclude each item | O(2^n) |
| Arithmetic sequence | triangular nested loops | O(n squared) |
| Geometric sequence | tree levels or repeated multiplication | powers of 2 |
| Modulo | divisibility, wraparound, cyclic arrays | remainder reasoning |

## Interview Questions and Answers

Use these as spoken practice. For each question, start with the concept, then give the short calculation or reasoning step an interviewer can follow.

### 1. Why is binary search O(log n)?

Binary search repeatedly cuts the remaining search space in half. After k steps, the remaining candidates are roughly n divided by 2 to the k.

```math
\frac{n}{2^k} = 1 \Rightarrow k = \log_2(n)
```

**Answer:** Binary search is O(log n) because each comparison removes half of the remaining candidates. It takes about log base 2 of n comparisons to reduce n items down to one candidate.

### 2. Convert binary 101101 to decimal.

A binary digit is worth a power of 2 based on its position from the right.

```math
101101_2 = 1 \cdot 2^5 + 0 \cdot 2^4 + 1 \cdot 2^3 + 1 \cdot 2^2 + 0 \cdot 2^1 + 1 \cdot 2^0
```

```math
32 + 8 + 4 + 1 = 45
```

**Answer:** 101101 in binary equals 45 in decimal.

### 3. How many subsets does a list of 5 items have?

Each item has two choices: included or excluded. Five independent yes/no choices produce 2 to the 5 outcomes.

```math
2^5 = 32
```

**Answer:** A list of 5 items has 32 subsets. If an algorithm must output every subset, O(2^n) time is usually unavoidable because the output itself has that many results.

### 4. How many permutations are there for 4 distinct items?

Permutations count orderings. The first slot has 4 choices, then 3, then 2, then 1.

```math
4! = 4 \cdot 3 \cdot 2 \cdot 1 = 24
```

**Answer:** There are 24 permutations. If a brute-force solution tries every ordering, its growth is factorial and becomes expensive very quickly.

### 5. What is the time complexity of this triangular nested loop?

This question tests whether you can turn loop shape into a count. The inner loop runs 0 times, then 1 time, then 2 times, up to n minus 1 times.

```python
def count_pairs(n: int) -> int:
    count = 0
    for i in range(n):
        for _ in range(i):
            count += 1
    return count
```

The total work is the arithmetic series from 0 to n minus 1.

```math
0 + 1 + 2 + \cdots + (n - 1) = \frac{n(n - 1)}{2}
```

**Answer:** The loop is O(n squared). Even though the exact count is half of n squared minus lower-order terms, Big-O keeps the dominant growth.

### 6. How many nodes are in a perfect binary tree of height 3?

A perfect binary tree doubles the node count at each level. Height 3 means levels 0, 1, 2, and 3.

```math
1 + 2 + 4 + 8 = 15
```

**Answer:** A perfect binary tree of height 3 has 15 nodes. In general, the node count grows like powers of 2, while the height grows like log n.

### 7. Why does primality checking only need to test divisors up to the square root?

Factors come in pairs. If both factors were greater than the square root, their product would be greater than n.

```math
a \cdot b = n
```

```math
a > \sqrt{n}\ \text{and}\ b > \sqrt{n}\Rightarrow a \cdot b > n
```

**Answer:** If n has any divisor larger than its square root, the paired divisor must be smaller than the square root. So testing divisors while `divisor * divisor <= n` is enough.

### 8. How does modulo help rotate an array index?

Modulo wraps an index back to the beginning once it passes the last valid position. This is the same idea as a clock wrapping after 12.

```python
def rotated_index(index: int, shift: int, length: int) -> int:
    return (index + shift) % length


assert rotated_index(4, 2, 5) == 1
```

**Answer:** Use `(index + shift) % length`. The modulo keeps the result inside the valid index range from 0 through length minus 1.

### 9. What is the difference between a subset and a permutation?

The key distinction is whether order matters. A subset is about choosing items; a permutation is about arranging items.

**Answer:** A subset asks, "Which elements are included?" A permutation asks, "In what order are the elements arranged?" Subsets usually imply 2 to the n growth, while permutations usually imply factorial growth.

### 10. When is exponential complexity acceptable in an interview solution?

Exponential complexity can be acceptable when the problem explicitly asks for an exponential-size output, or when the input constraints are small enough.

**Answer:** O(2^n) is reasonable for generating all subsets because there are 2 to the n subsets to return. It is less reasonable when the problem only asks for one value and a polynomial or logarithmic strategy exists.

## Practice

Work these by hand before coding. The point is to train recognition.

```python
assert binary_to_decimal("1011") == 11
assert divisions_by_two_until_one(32) == 5
assert factorial(4) == 24
assert len(subsets([1, 2, 3, 4])) == 16
assert triangular_work(10) == 55
assert perfect_binary_tree_nodes(4) == 31
assert sum_mod([13, 2, 24], 12) == 3
assert is_prime(29)
```

If these feel automatic, you know enough math to move deeper into data structures and algorithms.

## Sources

- User-provided pasted text, "Math for Technical Interviews", on 2026-06-10.
- Conversation with user on 2026-06-10.
- Python documentation, numeric types: https://docs.python.org/3/library/stdtypes.html#numeric-types-int-float-complex
- Python documentation, sorting HOWTO: https://docs.python.org/3/howto/sorting.html

## Related

- [Hash Tables](../../Data-Structures-and-Algorithms/05-hash-tables.md)
- [Two Pointers](../../Data-Structures-and-Algorithms/24-two-pointers.md)
- [Linked Lists](../../Data-Structures-and-Algorithms/03-linked-lists.md)
