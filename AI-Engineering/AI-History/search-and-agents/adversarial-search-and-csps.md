---
title: "Adversarial Search and CSPs"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# Adversarial Search and CSPs

[toc]

> **TL;DR:** Adversarial search extends goal-based search to zero-sum two-player games where an opponent actively minimizes the agent's utility — the minimax algorithm computes the optimal strategy by assuming the opponent plays perfectly. Alpha-beta pruning cuts irrelevant branches without changing the result, recovering up to O(b^(m/2)) savings. Constraint satisfaction problems (CSPs) reframe search over structured variable assignments, enabling powerful constraint propagation techniques (arc consistency, forward checking) that can collapse the search space before any backtracking is attempted.

## Vocabulary

**Game (adversarial)** — a search problem with two (or more) agents whose utility functions conflict; the standard formalization covers deterministic, perfect-information, zero-sum, two-player games.

**Zero-sum game** — one where the sum of all players' utilities is constant at every outcome; maximizing one player's utility exactly minimizes the other's.

**Ply** — one move by one player; a complete round (both players move) is two plies.

**MAX player** — the agent we are trying to find optimal play for; seeks to *maximize* the minimax value.

**MIN player** — the opponent; seeks to *minimize* the minimax value.

**Terminal state** — a state in which the game has ended; the utility function is defined.

**Utility (terminal)** — the numeric value assigned to a terminal state from MAX's perspective; typically +1 (win), 0 (draw), –1 (loss).

**Minimax value MINIMAX(s)** — the optimal utility achievable from state s against a perfect opponent.

**Evaluation function e(s)** — a heuristic estimate of the minimax value at a non-terminal state; used in h-minimax when full tree search is too expensive.

**Cutoff test** — a predicate that terminates search early (at a depth limit or quiescence condition), substituting an evaluation function for the true utility.

---

**CSP (Constraint Satisfaction Problem)** — a triple (X, D, C): X is a set of n variables, D is a set of n domains (one per variable), C is a set of constraints.

**Constraint** — a relation restricting allowable value combinations for a subset of variables.

**Unary constraint** — involves a single variable (e.g., SA ≠ green).

**Binary constraint** — involves a pair of variables (e.g., SA ≠ WA); the most common type; forms a *constraint graph*.

**Global constraint** — involves 3 or more variables (e.g., Alldiff, which asserts all variables take distinct values).

**Consistent assignment** — a partial assignment that violates no constraint.

**Complete assignment** — every variable has a value.

**Solution** — a complete, consistent assignment.

**Backtracking search (BTS)** — DFS on the CSP state space that assigns one variable at a time and checks constraints incrementally.

**Forward checking (FC)** — after each assignment, prune values from unassigned neighbors' domains that would violate a constraint with the just-assigned variable.

**Arc consistency (AC)** — a variable X_i is arc-consistent with X_j iff for every value in D_i there exists some value in D_j that satisfies the constraint between them.

**AC-3** — the standard arc-consistency algorithm; runs in O(cd^3) time for c constraints and domain size d.

**MRV (Minimum Remaining Values)** — heuristic: assign next the variable with the fewest remaining legal values ("fail-first").

**LCV (Least Constraining Value)** — heuristic: for a chosen variable, try first the value that eliminates the fewest values from remaining variables' domains.

---

## Intuition

### Adversarial search

In single-agent search, the environment is cooperative — states do not fight back. In a two-player game, the opponent's moves are adversarial: they will always choose the move that is worst for you. Minimax encodes this directly: MAX chooses the child with the highest value; MIN chooses the child with the lowest. The tree alternates MAX and MIN layers, propagating terminal utilities upward.

Alpha-beta pruning exploits a key observation: once MAX has found a move with value v, it will never accept a MIN node that can produce a value ≤ v (because MIN would steer there). Symmetrically, MIN will not accept a MAX node that can produce a value ≥ its current β bound. These bounds prune entire subtrees, cutting the effective branching factor in half under optimal node ordering.

**Figure:** minimax tree — terminal values propagate upward alternating max/min.

```
                MAX: choose max
              ┌──────┴──────┐
           MIN:3           MIN:2
         ┌──┴──┐         ┌──┴──┐
        3      5         2     9   (terminal utilities)
```

Root MAX = max(3, 2) = 3.

### CSPs

Standard search treats a state as an atomic blob. CSPs expose the internal *structure* of a state: it is a tuple of variable-value assignments, and each assignment must satisfy constraints. This structure enables two algorithmic weapons unavailable to plain DFS:

1. **Variable ordering heuristics** — choose which variable to assign next based on constraint information, not arbitrary order.
2. **Constraint propagation** — when a variable is assigned, propagate consequences to reduce other variables' domains, potentially detecting failure early (forward checking) or globally achieving arc consistency (AC-3).

## How it works

### Minimax algorithm

Minimax performs a depth-first traversal of the full game tree. At MAX nodes it returns the maximum child value; at MIN nodes the minimum. At terminal states it returns the utility directly. The algorithm is complete for finite game trees and optimal against a perfect opponent.

```python
from typing import Any, Callable, Optional

def minimax_value(
    state: Any,
    is_max: bool,
    terminal_test: Callable[[Any], bool],
    utility: Callable[[Any], float],
    actions: Callable[[Any], list[Any]],
    result: Callable[[Any, Any], Any],
) -> float:
    if terminal_test(state):
        return utility(state)
    values = [
        minimax_value(result(state, a), not is_max, terminal_test, utility, actions, result)
        for a in actions(state)
    ]
    return max(values) if is_max else min(values)

def minimax_decision(
    state: Any,
    terminal_test: Callable[[Any], bool],
    utility: Callable[[Any], float],
    actions: Callable[[Any], list[Any]],
    result: Callable[[Any, Any], Any],
) -> Any:
    """Return the best action for MAX from state."""
    return max(
        actions(state),
        key=lambda a: minimax_value(
            result(state, a), False, terminal_test, utility, actions, result
        ),
    )
```

**Properties:** Complete (finite game trees) — Yes. Optimal (vs. perfect opponent) — Yes. Time — O(b^m). Space — O(bm).

**Concrete example from lecture slides:** terminal values at leaves [2, 9, 7, 4, 8, 9, 3, 5].
- MIN layer: [min(2,9), min(7,4), min(8,9), min(3,5)] = [2, 4, 8, 3]
- MAX layer: [max(2,4), max(8,3)] = [4, 8]
- MIN root: min(4, 8) = 4
- MAX root: max(4) = 4... wait — with 8 leaves and b=2, the root is MAX selecting max over two MIN values = max(min(2,9), min(7,4)) = max(2,4) = 4. With b=2, m=3: root MAX = 4.

### Alpha-beta pruning

Alpha-beta maintains two bounds during the DFS: α (MAX's current lower bound — the best value MAX can guarantee so far) and β (MIN's current upper bound — the best value MIN can guarantee so far). At a MIN node, if the current value becomes ≤ α, prune (MAX would never allow this subtree). At a MAX node, if the current value becomes ≥ β, prune (MIN would never allow this subtree).

```python
def alpha_beta(
    state: Any,
    is_max: bool,
    alpha: float,
    beta: float,
    terminal_test: Callable[[Any], bool],
    utility: Callable[[Any], float],
    actions: Callable[[Any], list[Any]],
    result: Callable[[Any, Any], Any],
) -> float:
    if terminal_test(state):
        return utility(state)
    if is_max:
        v = float("-inf")
        for a in actions(state):
            v = max(v, alpha_beta(
                result(state, a), False, alpha, beta,
                terminal_test, utility, actions, result
            ))
            if v >= beta:
                return v   # beta cutoff: MIN would avoid this
            alpha = max(alpha, v)
        return v
    else:
        v = float("inf")
        for a in actions(state):
            v = min(v, alpha_beta(
                result(state, a), True, alpha, beta,
                terminal_test, utility, actions, result
            ))
            if v <= alpha:
                return v   # alpha cutoff: MAX would avoid this
            beta = min(beta, v)
        return v

def alpha_beta_decision(state: Any, ...) -> Any:
    return max(
        actions(state),
        key=lambda a: alpha_beta(
            result(state, a), False, float("-inf"), float("inf"),
            terminal_test, utility, actions, result
        ),
    )
```

> [!IMPORTANT]
> Alpha-beta returns the **same** minimax value as plain minimax — it never changes the answer, only the work. The classic illustration: in the tree max(min(3,12,8), min(2,X,Y), min(14,5,2)), the root value is 3 regardless of X and Y because MAX locks in 3 from the first MIN subtree, and can verify no other branch improves on it without ever evaluating X and Y.

Under optimal move ordering (best moves examined first), alpha-beta reduces the effective branching factor from b to √b, handling game trees of depth 2m in the same time minimax handles depth m. For chess (b ≈ 35, m ≈ 100 plies for full game), this is the difference between tractable and astronomically intractable.

### h-Minimax (heuristic minimax)

Real games (chess, Go) have branching factors and depths too large for full minimax. h-Minimax replaces the terminal utility with an *evaluation function* e(s) at a depth cutoff or *quiescence* condition (no captures or checks pending). The cutoff test replaces the terminal test. The resulting search quality depends entirely on the quality of e(s).

Deep Blue (1997) used a linear combination of ~8000 handcrafted features in its evaluation function. AlphaGo (2016) replaced the handcrafted evaluation with a value network trained by self-play, essentially learning a neural-network e(s).

### CSP: Backtracking Search

Backtracking search for CSPs is DFS with two modifications: (1) variables are assigned one at a time (not all at once), and (2) constraints are checked *as assignments are made* (not only at leaf nodes). The key insight is that variable-assignment order is *commutative*: {WA=red, NT=green} is the same partial assignment as {NT=green, WA=red}, so BFS generates exponentially more nodes for no benefit.

```python
from typing import Any, Optional

def backtrack(
    assignment: dict[str, Any],
    variables: list[str],
    domains: dict[str, list[Any]],
    constraints: list[tuple],
) -> Optional[dict[str, Any]]:
    if len(assignment) == len(variables):
        return assignment
    var = select_unassigned_variable(variables, assignment, domains)
    for value in order_domain_values(var, assignment, domains, constraints):
        if is_consistent(var, value, assignment, constraints):
            assignment[var] = value
            result = backtrack(assignment, variables, domains, constraints)
            if result is not None:
                return result
            del assignment[var]
    return None

def select_unassigned_variable(
    variables: list[str],
    assignment: dict[str, Any],
    domains: dict[str, list[Any]],
) -> str:
    # MRV: choose variable with fewest remaining legal values
    unassigned = [v for v in variables if v not in assignment]
    return min(unassigned, key=lambda v: len(domains[v]))

def is_consistent(
    var: str,
    value: Any,
    assignment: dict[str, Any],
    constraints: list[tuple],
) -> bool:
    """Check that (var=value) is consistent with current assignment."""
    for (v1, v2, relation) in constraints:
        if v1 == var and v2 in assignment:
            if not relation(value, assignment[v2]):
                return False
        if v2 == var and v1 in assignment:
            if not relation(assignment[v1], value):
                return False
    return True
```

### Forward Checking

After assigning variable X_i = v, forward checking removes every value from the domain of every unassigned variable X_j that conflicts with v (via the constraint between X_i and X_j). If any domain becomes empty, fail immediately without expanding further. This detects guaranteed failures one level ahead, avoiding many useless tree branches.

**Australia map-coloring trace:** After assigning WA=red, forward checking removes red from NT and SA's domains. After NT=green, it removes green from Q and SA. After Q=red (only legal value for Q), it removes red from NSW. At this point SA has only blue left — assignment is forced.

> [!NOTE]
> Forward checking is *not* arc consistency. FC only checks arcs between the newly assigned variable and unassigned variables. Arc consistency (AC-3) propagates constraints between all unassigned variables, detecting failures FC would miss. Example: FC won't detect that NT and SA must be different colors even if neither has been assigned yet.

### AC-3 Algorithm

AC-3 achieves arc consistency by maintaining a queue of arcs (X_i, X_j). For each arc, it removes from D_i any value with no supporting value in D_j. When a domain shrinks, all arcs pointing to that variable are re-queued (since their consistency may now be violated). AC-3 runs in O(c · d^3) time where c = number of constraints and d = maximum domain size.

```python
from collections import deque

def ac3(
    variables: list[str],
    domains: dict[str, list[Any]],
    constraints: dict[tuple[str, str], Any],
) -> bool:
    """Enforce arc consistency. Modifies domains in-place. Returns False if unsatisfiable."""
    queue: deque[tuple[str, str]] = deque(constraints.keys())
    while queue:
        xi, xj = queue.popleft()
        if revise(xi, xj, domains, constraints):
            if not domains[xi]:
                return False  # domain wipeout — no solution
            # Re-queue all arcs pointing to xi (excluding xi->xj)
            for xk in variables:
                if xk != xj and (xk, xi) in constraints:
                    queue.append((xk, xi))
    return True

def revise(
    xi: str,
    xj: str,
    domains: dict[str, list[Any]],
    constraints: dict[tuple[str, str], Any],
) -> bool:
    """Remove values from D_xi that have no support in D_xj. Returns True if revised."""
    relation = constraints[(xi, xj)]
    revised = False
    for x in list(domains[xi]):
        if not any(relation(x, y) for y in domains[xj]):
            domains[xi].remove(x)
            revised = True
    return revised
```

## Math

**Minimax value recurrence:**

```math
\text{MINIMAX}(s) =
\begin{cases}
\text{UTILITY}(s) & \text{if } \text{TERMINAL-TEST}(s) \\
\max_{a \in \text{ACTIONS}(s)} \text{MINIMAX}(\text{RESULT}(s,a)) & \text{if MAX's turn} \\
\min_{a \in \text{ACTIONS}(s)} \text{MINIMAX}(\text{RESULT}(s,a)) & \text{if MIN's turn}
\end{cases}
```

**Alpha-beta pruning conditions:**

At a MIN node v: prune if v ≤ α (MAX already has a better option above)

At a MAX node v: prune if v ≥ β (MIN already has a better option above)

**Alpha-beta savings:** With perfect move ordering, alpha-beta reduces time complexity from O(b^m) to O(b^(m/2)), effectively doubling the searchable depth.

**CSP complexity:** A CSP with n variables each with domain size d has d^n complete assignments. With backtracking + MRV + forward checking, the practical complexity is much lower, but worst-case remains exponential in the number of variables (NP-hard in general).

**AC-3 complexity:** O(c · d^3) time, O(c) space (the arc queue). Here c = number of arcs = O(n^2) for binary CSPs, so the overall time is O(n^2 · d^3).

## Real-world example

Solving the Australia map-coloring problem with backtracking + MRV + LCV + forward checking:

```python
from typing import Any, Optional

# Variables, domains, constraints for Australia map coloring
variables: list[str] = ["WA", "NT", "Q", "NSW", "V", "SA", "T"]
domains: dict[str, list[str]] = {v: ["red", "green", "blue"] for v in variables}

# Neighbors (adjacencies) — SA is adjacent to all except T
neighbors: dict[str, list[str]] = {
    "WA": ["NT", "SA"],
    "NT": ["WA", "Q", "SA"],
    "Q": ["NT", "NSW", "SA"],
    "NSW": ["Q", "V", "SA"],
    "V": ["NSW", "SA"],
    "SA": ["WA", "NT", "Q", "NSW", "V"],
    "T": [],
}

def different(a: str, b: str) -> bool:
    return a != b

def csp_map_solve() -> Optional[dict[str, str]]:
    """Solve Australia map coloring via backtracking with MRV and forward checking."""

    def forward_check(
        var: str, value: str, domains_copy: dict[str, list[str]]
    ) -> bool:
        """Prune values from neighbors' domains. Returns False if any domain empties."""
        for neighbor in neighbors[var]:
            if value in domains_copy[neighbor]:
                domains_copy[neighbor] = [
                    v for v in domains_copy[neighbor] if v != value
                ]
                if not domains_copy[neighbor]:
                    return False
        return True

    def backtrack(
        assignment: dict[str, str],
        domains_copy: dict[str, list[str]],
    ) -> Optional[dict[str, str]]:
        if len(assignment) == len(variables):
            return assignment
        # MRV: pick unassigned variable with smallest domain
        unassigned = [v for v in variables if v not in assignment]
        var = min(unassigned, key=lambda v: len(domains_copy[v]))
        # LCV: order values by how few constraints they impose on neighbors
        def count_ruled_out(val: str) -> int:
            return sum(
                1 for nb in neighbors[var]
                if nb not in assignment and val in domains_copy[nb]
            )
        for value in sorted(domains_copy[var], key=count_ruled_out):
            # Consistency check
            if all(
                different(value, assignment[nb])
                for nb in neighbors[var]
                if nb in assignment
            ):
                assignment[var] = value
                new_domains = {v: list(d) for v, d in domains_copy.items()}
                if forward_check(var, value, new_domains):
                    result = backtrack(assignment, new_domains)
                    if result is not None:
                        return result
                del assignment[var]
        return None

    return backtrack({}, {v: list(d) for v, d in domains.items()})

solution = csp_map_solve()
print(solution)
# Expected: {'WA': 'red', 'NT': 'green', 'Q': 'red', 'NSW': 'green', 'V': 'red', 'SA': 'blue', 'T': 'red'}
```

> [!TIP]
> SA is the bottleneck variable in the Australia problem — it is adjacent to 5 of 6 other regions. MRV will naturally select SA early once enough neighbors are assigned, and forward checking will rapidly detect domain wipeouts on SA, pruning unpromising branches before they are explored. This is the "fail-first" principle in action.

## In practice

**Alpha-beta's practical impact is move-ordering dependent.** Worst-case ordering (worst moves first) gives no benefit over minimax. In chess engines, iterative deepening + transposition tables reuse results from shallower searches to order moves better at deeper levels, approaching the theoretical O(b^(m/2)) improvement.

**The horizon effect is the key failure mode of h-minimax.** The evaluation function at depth d may assign a high score to a position that is immediately followed by a capture at depth d+1 (just over the horizon). *Quiescence search* addresses this by extending search past captures and checks until a quiet position is reached, even if it exceeds the nominal depth limit.

**CSPs benefit enormously from constraint propagation as preprocessing.** For Sudoku, AC-3 alone solves the majority of newspaper-difficulty puzzles without any backtracking. Only the hardest "diabolical" Sudoku instances require search after propagation exhausts. The same principle applies to circuit layout, scheduling, and resource allocation — propagation should always precede search.

> [!CAUTION]
> Global constraint handling matters in real CSP solvers. A naive binary encoding of Alldiff over n variables creates O(n^2) binary constraints. Dedicated Alldiff propagation algorithms run in O(n log n) via bipartite matching, with exponentially better pruning. Always use specialized global constraint handlers in production CSP solvers (e.g., OR-Tools, Choco, MiniZinc).

**AI milestones cited in lectures:** Chinook (1994) solved checkers; Deep Blue (1997) defeated Kasparov at chess; AlphaGo (2016) defeated Lee Sedol at Go using Monte Carlo Tree Search + neural evaluation functions rather than classic minimax.

## Pitfalls

- **Wrong belief: alpha-beta changes the minimax value.** Correction: alpha-beta is a pure optimization of minimax — it prunes branches that cannot affect the final answer. The returned value is identical to what plain minimax would return.

- **Wrong belief: MRV always reduces backtracking.** Correction: MRV is a heuristic, not a theorem. In adversarial or constrained problems without much structure, it may offer little benefit. Its power comes from early detection of domain wipeouts — it fails-first, which is valuable precisely when failure is common.

- **Wrong belief: forward checking achieves arc consistency.** Correction: forward checking only propagates from the most recently assigned variable to its neighbors. Arc consistency propagates transitively through the constraint graph. Example: with WA=red, FC removes red from NT and SA. But it does not ensure consistency between NT and Q, or between SA and NSW.

- **Wrong belief: a bigger evaluation function is always better for h-minimax.** Correction: evaluation functions that are complex have two failure modes — they may be slow (reducing depth searchable in fixed time) and may be less accurate (more parameters to overfit or misestimate). Empirically, Deep Blue's 8000-feature evaluation was competitive, but AlphaGo's neural evaluation *learned* features from self-play, outperforming handcrafted alternatives.

- **Wrong belief: minimax assumes the opponent plays randomly.** Correction: minimax assumes the opponent plays *optimally* (worst-case for MAX). Against a weak opponent, minimax may not actually maximize real-game performance (it is not Bayes-optimal against a non-minimax player). Expectiminimax handles stochastic opponents; probabilistic opponent modeling handles strategic deviation.

## Exercises

### Exercise 1

Given the game tree below (MAX moves at root, MIN at depth 1, terminal values at depth 2), trace alpha-beta pruning with initial α = −∞, β = +∞:

```
          MAX
       /        \
    MIN           MIN
   / \            / \
  3   8          2   9
```

Which nodes are pruned? What is the minimax value?

#### Solution 1

Traverse left-to-right, depth-first.

1. Visit left MIN node. Evaluate left child: 3. MIN value so far = 3. Update β = 3 at this MIN node.
2. Evaluate right child: 8. min(3, 8) = 3. Left MIN node returns 3.
3. Back at MAX root: α = 3 (best seen so far for MAX).
4. Visit right MIN node. Evaluate left child: 2. min(∞, 2) = 2. At this MIN node, v = 2 ≤ α = 3. **Alpha cutoff: prune the right child (9).**
5. Right MIN node returns 2.
6. MAX root: max(3, 2) = **3**.

Pruned node: the right child (9) of the right MIN node.
Minimax value: **3**.

---

### Exercise 2

Define a CSP for the 4-queens problem (place 4 non-attacking queens on a 4×4 board, one per column). Give variables, domains, and constraints.

#### Solution 2

**Variables:** Q1, Q2, Q3, Q4 — one variable per column; Q_i = row of the queen in column i.

**Domains:** D_i = {1, 2, 3, 4} for all i.

**Constraints (for all pairs i < j):**
- Not in the same row: Q_i ≠ Q_j
- Not on the same diagonal: |Q_i − Q_j| ≠ |i − j|

That is: Q_i ≠ Q_j AND Q_i − Q_j ≠ i − j AND Q_j − Q_i ≠ i − j.

These collapse to a single binary constraint for each pair (i, j): Q_i ≠ Q_j AND |Q_i − Q_j| ≠ j − i.

**Number of constraints:** C(4,2) = 6 binary constraints.

**One solution:** Q1=2, Q2=4, Q3=1, Q4=3 (queens at rows 2, 4, 1, 3 in columns 1–4).

---

### Exercise 3

Apply one pass of arc consistency (AC-3 style) to the following mini-CSP:
- Variables: X ∈ {1,2,3}, Y ∈ {1,2,3}
- Constraint: X < Y

Start with arc (X, Y) and then (Y, X). What are the resulting domains?

#### Solution 3

**Arc (X, Y): for each value of X, there must exist some value in Y such that X < Y.**
- X=1: Y can be 2 or 3. OK — keep X=1.
- X=2: Y can be 3. OK — keep X=2.
- X=3: No Y in {1,2,3} satisfies 3 < Y. **Remove X=3.**

After revise(X, Y): D(X) = {1, 2}, D(Y) = {1, 2, 3}.

**Arc (Y, X): for each value of Y, there must exist some value in X such that X < Y.**
- Y=1: No X in {1, 2} satisfies X < 1. **Remove Y=1.**
- Y=2: X can be 1. OK — keep Y=2.
- Y=3: X can be 1 or 2. OK — keep Y=3.

After revise(Y, X): D(X) = {1, 2}, D(Y) = {2, 3}.

**Result:** D(X) = {1, 2}, D(Y) = {2, 3}. The arc (X, Y) was revised (D(X) shrank), so we would re-queue all arcs pointing to X — but since there are only two variables here, AC-3 terminates. Both remaining arcs are already arc-consistent: X=1 is supported by Y=2 or Y=3; X=2 is supported by Y=3; Y=2 is supported by X=1; Y=3 is supported by X=1 or X=2.

---

## Sources

- Russell, S. & Norvig, P. *Artificial Intelligence: A Modern Approach*, 4th ed. Chapters 5–6. Pearson, 2020.
- Lecture slides: air(5).pdf — Adversarial Search; air(6).pdf — CSPs (University course materials, 2025).
- Schaeffer, J. et al. "Chinook: world man-machine checkers champion." *AI Magazine* 17(1):21–29, 1996.
- Campbell, M. et al. "Deep Blue." *Artificial Intelligence* 134(1-2):57–83, 2002.
- Silver, D. et al. "Mastering the game of Go with deep neural networks and tree search." *Nature* 529:484–489, 2016.
- Mackworth, A. "Consistency in networks of relations." *Artificial Intelligence* 8(1):99–118, 1977. (AC-3 predecessor)
- Conversation with user, 2026-05-19.

## Related

- [Intelligent Agents](./intelligent-agents.md)
- [Uninformed and Informed Search](./uninformed-and-informed-search.md)
- [Logical Agents and Knowledge Representation](./4-logical-agents-and-knowledge-representation.md)
