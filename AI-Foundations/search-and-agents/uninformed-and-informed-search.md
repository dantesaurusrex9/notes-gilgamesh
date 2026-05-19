---
title: "Uninformed and Informed Search"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# Uninformed and Informed Search

[toc]

> **TL;DR:** Search formalizes goal-based agents as graph traversal: states are nodes, actions are edges, and a solution is a path from the initial state to any goal state. Uninformed algorithms (BFS, DFS, UCS, IDS) exploit only the graph structure; informed algorithms (A*, greedy best-first) additionally exploit a *heuristic* — a domain-specific estimate of remaining cost. A* with an admissible heuristic is the gold standard: it is complete, optimal, and provably expands the fewest nodes among algorithms using the same heuristic.

## Vocabulary

**State space** — the set of all states reachable from the initial state by any sequence of actions; represented as a directed graph.

**Initial state** — the state in which the agent begins.

**Actions(s)** — the set of actions available in state s.

**Result(s, a)** — the state produced by applying action a in state s (the successor function / transition model).

**Goal test** — a predicate that returns True iff a state is a goal state.

**Path cost** — a function g(path) assigning a numeric cost to a sequence of actions; often the sum of individual step costs.

**Solution** — a path from the initial state to a goal state; an *optimal solution* minimizes path cost.

**Branching factor (b)** — maximum number of successors of any node.

**Depth (d)** — depth of the shallowest goal node in the search tree.

**Maximum depth (m)** — maximum depth of any path in the search space (may be infinite).

---

**Frontier** — the set of nodes generated but not yet expanded (the "open list").

**Explored set** — the set of nodes already expanded (the "closed list"); used to avoid re-expansion.

**Complete** — an algorithm is complete if it is guaranteed to find a solution when one exists.

**Optimal** — an algorithm is optimal if it is guaranteed to return a solution with minimum path cost.

---

**Heuristic h(n)** — an estimate of the cost from node n to the nearest goal; must be non-negative with h(goal) = 0.

**Admissible heuristic** — one that never *overestimates* the true cost: for all n, h(n) ≤ h*(n), where h*(n) is the true optimal cost from n.

**Consistent (monotone) heuristic** — for every node n and successor n' via action a: h(n) ≤ c(n, a, n') + h(n'). Consistency implies admissibility; the converse does not hold in general.

**f(n) = g(n) + h(n)** — the evaluation function in A*: g(n) is the cost from start to n, h(n) is the heuristic estimate from n to goal.

---

## Intuition

Think of uninformed search as walking through a dark room: you know the layout (the graph), but you have no flashlight. BFS feels the walls outward in concentric rings, guaranteeing you find the nearest exit — but carrying all those rings in memory is expensive. DFS goes straight down one corridor as far as it can, then backtracks — memory-cheap but it might walk past the exit down a dead-end corridor of infinite length. UCS generalizes BFS to weighted edges: always extend the cheapest frontier node. IDS gets BFS's completeness guarantee with DFS's memory footprint by running DFS repeatedly with increasing depth limits.

Now give the agent a flashlight — a heuristic h(n) that estimates remaining distance. Greedy best-first follows only the flashlight, ignoring how far it has already walked: fast in practice, but not optimal. A* combines both: it never expands a node unless it must (because the true cost through that node could be optimal). With an admissible h, A* is provably optimal and expands the minimum number of nodes among all optimal algorithms with the same h.

**Figure:** relationship between search algorithms on two axes — use of cost information (g) and use of heuristic information (h).

```
           No g         Uses g
          ┌─────────────┬──────────────────┐
No h      │     DFS     │       UCS        │
          ├─────────────┼──────────────────┤
Uses h    │  Greedy-BF  │       A*         │
          └─────────────┴──────────────────┘
```

## How it works

All search algorithms share the same skeleton: initialize a frontier with the start node; repeatedly remove the best node by some criterion; check goal; if not goal, expand and add children to frontier. The algorithms differ only in what "best" means and whether an explored set is maintained.

### BFS — Breadth-First Search

BFS uses a FIFO queue as its frontier, expanding nodes in order of increasing depth. It is complete (always finds a solution if one exists, as long as b and d are finite) and optimal when all step costs are equal (uniform cost 1). The fatal weakness is memory: at depth d with branching factor b, the frontier holds O(b^d) nodes. The course slides give a visceral example: at depth 10 with b=10, the frontier requires ~10 terabytes; at depth 16, ~10 exabytes. BFS is the right choice only for shallow goal depths or when memory is not a constraint.

```python
from collections import deque
from typing import Any, Callable, Optional

def bfs(
    initial: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[Any]],
) -> Optional[list[Any]]:
    """Returns list of states on the solution path, or None."""
    if goal_test(initial):
        return [initial]
    frontier: deque[list[Any]] = deque([[initial]])
    explored: set[Any] = set()
    while frontier:
        path = frontier.popleft()
        state = path[-1]
        if state in explored:
            continue
        explored.add(state)
        for child in successors(state):
            new_path = path + [child]
            if goal_test(child):
                return new_path
            if child not in explored:
                frontier.append(new_path)
    return None
```

| Property | BFS |
| :--- | :--- |
| Complete | Yes (if b and d finite) |
| Optimal | Yes (uniform step cost) |
| Time | O(b^d) |
| Space | O(b^d) — the frontier dominates |

### DFS — Depth-First Search

DFS uses a LIFO stack (or recursion), always expanding the deepest node first. Memory is excellent: O(bm) where m is the maximum depth — the stack holds at most one path from root to leaf. The catastrophic weakness is completeness: in infinite-depth spaces (or spaces with cycles), DFS may never return. Even with a cycle check, it can find a very long non-optimal solution. Use DFS when memory is tight, the depth is bounded, and any solution (not necessarily optimal) will do.

```python
from typing import Any, Callable, Optional

def dfs(
    state: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[Any]],
    explored: Optional[set[Any]] = None,
    path: Optional[list[Any]] = None,
) -> Optional[list[Any]]:
    if explored is None:
        explored = set()
    if path is None:
        path = [state]
    if goal_test(state):
        return path
    explored.add(state)
    for child in successors(state):
        if child not in explored:
            result = dfs(child, goal_test, successors, explored, path + [child])
            if result is not None:
                return result
    return None
```

| Property | DFS |
| :--- | :--- |
| Complete | No (infinite spaces) — Yes with depth bound |
| Optimal | No |
| Time | O(b^m) |
| Space | O(bm) |

### IDS — Iterative Deepening Search

IDS runs depth-limited DFS repeatedly, incrementing the depth limit by 1 on each iteration. It inherits BFS's completeness and optimality (uniform cost) while paying only DFS-level memory O(bd). The apparent redundancy of re-expanding nodes at every depth costs little in practice: most work happens at the deepest limit (the last frontier is ~b^d nodes, all previous depths together are only ~b^(d-1)/(b-1) ≈ b^d/b nodes). IDS is the preferred uninformed algorithm for large state spaces.

```python
from typing import Any, Callable, Optional

def dls(
    state: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[Any]],
    limit: int,
    path: Optional[list[Any]] = None,
) -> Optional[list[Any]]:
    if path is None:
        path = [state]
    if goal_test(state):
        return path
    if limit == 0:
        return None
    for child in successors(state):
        if child not in path:  # avoid cycles on current path
            result = dls(child, goal_test, successors, limit - 1, path + [child])
            if result is not None:
                return result
    return None

def ids(
    initial: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[Any]],
    max_depth: int = 1000,
) -> Optional[list[Any]]:
    for depth in range(max_depth + 1):
        result = dls(initial, goal_test, successors, depth)
        if result is not None:
            return result
    return None
```

### UCS — Uniform-Cost Search

UCS generalizes BFS to non-uniform step costs: it expands the node with the lowest cumulative path cost g(n) by using a min-priority queue as its frontier. It is complete and optimal for non-negative step costs. When all step costs are equal, UCS degenerates to BFS.

> [!IMPORTANT]
> UCS requires **non-negative** step costs. A single negative edge can cause UCS to cycle and never terminate. Bellman-Ford is the correct algorithm for graphs with negative-cost edges.

```python
import heapq
from typing import Any, Callable, Optional

def ucs(
    initial: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[tuple[Any, float]]],  # (state, cost)
) -> Optional[tuple[float, list[Any]]]:
    # heap: (cumulative_cost, unique_id, path)
    counter = 0
    heap: list[tuple[float, int, list[Any]]] = [(0.0, counter, [initial])]
    explored: dict[Any, float] = {}
    while heap:
        cost, _, path = heapq.heappop(heap)
        state = path[-1]
        if state in explored and explored[state] <= cost:
            continue
        explored[state] = cost
        if goal_test(state):
            return cost, path
        for child, step_cost in successors(state):
            new_cost = cost + step_cost
            if child not in explored or explored[child] > new_cost:
                counter += 1
                heapq.heappush(heap, (new_cost, counter, path + [child]))
    return None
```

### Greedy Best-First Search

Greedy best-first sorts the frontier by h(n) alone, ignoring g(n). It races toward the goal as indicated by the heuristic. This makes it very fast in practice when the heuristic is good — it found Bucharest from Arad in one fewer step than A* in the Romania example. But it is neither complete (can follow a heuristic into a dead end) nor optimal (ignores path cost).

### A* Search

A* evaluates each node with f(n) = g(n) + h(n), the estimated total path cost through n. With an admissible heuristic, A* is complete and optimal. The intuitive argument: if A* selects a goal node G, then f(G) = g(G) (since h(goal) = 0). Any unexpanded node n on an optimal path has f(n) = g(n) + h(n) ≤ g(n) + h*(n) = f(G*) ≤ f(G). So G would not be selected before G* — contradiction.

```python
import heapq
from typing import Any, Callable, Optional

def astar(
    initial: Any,
    goal_test: Callable[[Any], bool],
    successors: Callable[[Any], list[tuple[Any, float]]],
    heuristic: Callable[[Any], float],
) -> Optional[tuple[float, list[Any]]]:
    counter = 0
    # heap: (f, unique_id, g, path)
    heap: list[tuple[float, int, float, list[Any]]] = [
        (heuristic(initial), counter, 0.0, [initial])
    ]
    explored: dict[Any, float] = {}
    while heap:
        f, _, g, path = heapq.heappop(heap)
        state = path[-1]
        if state in explored and explored[state] <= g:
            continue
        explored[state] = g
        if goal_test(state):
            return g, path
        for child, step_cost in successors(state):
            new_g = g + step_cost
            if child not in explored or explored[child] > new_g:
                new_f = new_g + heuristic(child)
                counter += 1
                heapq.heappush(heap, (new_f, counter, new_g, path + [child]))
    return None
```

> [!NOTE]
> The course slides trace A* on the Romania road map: starting from Arad, the algorithm expands Sibiu, Rimnicu Vilcea, Fagaras, Pitesti, and finally Bucharest, finding the optimal 418-km path. The path via Fagaras (direct) looks tempting to greedy search but costs 450 km — A* avoids it because g(Fagaras) + h(Fagaras) > g(Pitesti) + h(Pitesti).

## Math

Let h*(n) denote the true (oracle) cost from node n to the nearest goal.

**Admissibility:** h is admissible iff for all n: h(n) ≤ h*(n).

**Consistency:** h is consistent iff for all n, n' (successor of n via action a):

```math
h(n) \leq c(n, a, n') + h(n')
```

This is a triangle inequality: the estimate at n cannot exceed the edge cost plus the estimate at n'.

**A* optimality theorem:** If h is admissible, then A* (with re-expansion when a better path is found via the explored set) returns an optimal solution.

**Proof sketch:** Suppose A* returns suboptimal goal G_s with g(G_s) > C* (optimal cost). There must be a node n on an optimal path that is on the frontier when G_s is chosen. Since h is admissible, f(n) = g(n) + h(n) ≤ C*. But A* selects the node with minimum f, so f(G_s) ≤ f(n) ≤ C*. Yet f(G_s) = g(G_s) + h(G_s) = g(G_s) > C*. Contradiction.

**8-puzzle heuristics (from the course slides):**

```math
h_1 = \text{number of misplaced tiles}
```

```math
h_2 = \sum_{i} \text{Manhattan distance of tile } i \text{ from its goal position}
```

For the example in the slides: h_1 = 8 (all tiles misplaced), h_2 = 18 (sum of Manhattan distances). Both are admissible. h_2 dominates h_1: h_2(n) ≥ h_1(n) for all n, so A* with h_2 expands fewer nodes.

**Algorithm comparison table:**

| Algorithm | Complete | Optimal | Time | Space |
| :--- | :---: | :---: | :---: | :---: |
| BFS | Yes | Yes (uniform) | O(b^d) | O(b^d) |
| DFS | No | No | O(b^m) | O(bm) |
| IDS | Yes | Yes (uniform) | O(b^d) | O(bd) |
| UCS | Yes | Yes | O(b^(C*/ε)) | O(b^(C*/ε)) |
| Greedy | No | No | O(b^m) | O(b^m) |
| A* | Yes | Yes (admissible h) | Exponential | O(b^d) — all nodes |

## Real-world example

The 8-puzzle: a 3×3 grid with tiles 1–8 and one blank. The task is to reach the goal configuration from a scrambled state.

```python
import heapq
from typing import Optional

State = tuple[int, ...]

GOAL: State = (1, 2, 3, 4, 5, 6, 7, 8, 0)  # 0 = blank

def manhattan_distance(state: State) -> int:
    """h2: sum of Manhattan distances of all tiles from their goal positions."""
    total = 0
    for i, tile in enumerate(state):
        if tile == 0:
            continue
        goal_idx = GOAL.index(tile)
        row_curr, col_curr = divmod(i, 3)
        row_goal, col_goal = divmod(goal_idx, 3)
        total += abs(row_curr - row_goal) + abs(col_curr - col_goal)
    return total

def get_successors(state: State) -> list[tuple[State, float]]:
    """Generate all states reachable in one move."""
    blank = state.index(0)
    row, col = divmod(blank, 3)
    moves = []
    for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
        nr, nc = row + dr, col + dc
        if 0 <= nr < 3 and 0 <= nc < 3:
            new_blank = nr * 3 + nc
            new_state = list(state)
            new_state[blank], new_state[new_blank] = new_state[new_blank], new_state[blank]
            moves.append((tuple(new_state), 1.0))
    return moves

# Solve a scrambled 8-puzzle with A* + Manhattan distance
initial: State = (7, 2, 4, 5, 0, 6, 8, 3, 1)
result = astar(
    initial=initial,
    goal_test=lambda s: s == GOAL,
    successors=get_successors,
    heuristic=manhattan_distance,
)
if result:
    cost, path = result
    print(f"Solved in {len(path)-1} moves, cost={cost:.0f}")
else:
    print("No solution found")
```

> [!TIP]
> For the 8-puzzle, the effective branching factor of A* with Manhattan distance is approximately 1.24, versus about 2.67 for A* with misplaced tiles (h_1). The difference compounds exponentially: for 20-move solutions, A* with h_2 expands roughly 6,400 nodes vs. ~450,000 with h_1. Always prefer the heuristic with the highest admissible values — it dominates and reduces search effort.

## In practice

**A*'s space problem is real.** The formal analysis says A* keeps all generated nodes in memory (because any could be revisited on an optimal path). For state spaces where the depth of the optimal solution is large, this is prohibitive. Practical variants include IDA* (iterative deepening A*, uses O(bd) space by re-expanding), RBFS (recursive best-first search), and SMA* (simplified memory-bounded A*). Most production planners use one of these, not vanilla A*.

**Heuristic quality has exponential leverage.** Every unit of slack in h(n) (the gap h*(n) - h(n)) can cause A* to expand exponentially more nodes. Relaxed-problem heuristics — derived by removing constraints from the original problem — are the standard construction. For the 8-puzzle, Manhattan distance comes from removing the constraint that tiles cannot move through each other. For many planning problems, the FF heuristic (from the Fast-Forward planner) uses delete-relaxation to derive admissible estimates at scale.

**BFS's memory horror is pedagogically important.** The slide numbers (depth 10 = 10 terabytes, depth 16 = 10 exabytes) show why BFS is essentially unusable for any nontrivial state space. IDS's memory advantage (depth 16 with BFS vs. 156 kilobytes with DFS) is not just academic — it is why IDS (not BFS) is the baseline uninformed algorithm in practice.

> [!WARNING]
> Greedy best-first search is not admissible — it can find suboptimal paths and can get stuck in dead ends when the heuristic is misleading. Do not use it when solution quality matters. Use A* instead.

## Pitfalls

- **Wrong belief: admissibility and consistency are the same condition.** Correction: consistency implies admissibility, but not vice versa. An inconsistent but admissible heuristic can cause A* to re-expand nodes if using the explored set naively; consistent heuristics guarantee monotonically non-decreasing f values along any path, so re-expansion is never needed.

- **Wrong belief: IDS wastes time re-expanding nodes.** Correction: the redundant work at levels above d is at most a constant fraction of the total work (1/(b-1) overhead for large b). At b=10, IDS does at most 11% more work than BFS while using O(bd) space instead of O(b^d).

- **Wrong belief: a higher h(n) is always better.** Correction: h must remain admissible. An inadmissible heuristic that overestimates can cause A* to miss the optimal solution. The trade-off is: inadmissible weighted A* (using w·h(n) with w > 1) runs faster and finds w-suboptimal solutions — useful in practice when time matters more than exact optimality.

- **Wrong belief: UCS is the same as Dijkstra's algorithm.** Correction: UCS and Dijkstra are essentially equivalent, but UCS is typically presented as a tree search with lazy duplicate detection, while Dijkstra is a graph algorithm that initializes all nodes. In a search context with a goal test and early termination, UCS stops when it pops the goal; Dijkstra typically processes all nodes.

- **Wrong belief: DFS cannot be used for optimality.** Correction: DFS can find optimal solutions if the search space is a tree (no cycles) and all paths to the goal have the same cost. Outside those conditions, DFS is not optimal.

## Exercises

### Exercise 1

Trace BFS on the following undirected graph (edges have unit cost) starting from node S with goal G.
```
S -- A -- B
|         |
C -- D -- G
```

List the order of node expansion and the solution path.

#### Solution 1

BFS frontier (FIFO queue), explored set:

1. Initialize: frontier = [S], explored = {}
2. Pop S, expand: frontier = [A, C], explored = {S}
3. Pop A, expand: frontier = [C, B], explored = {S, A}  (S already explored)
4. Pop C, expand: frontier = [B, D], explored = {S, A, C}
5. Pop B, expand: frontier = [D, G], explored = {S, A, C, B}
6. Pop D, expand: frontier = [G, G], explored = {S, A, C, B, D}  (G added from both B and D)
7. Pop G — **goal test passes.**

Expansion order: S, A, C, B, D, G.

Solution path: S → A → B → G (length 3) or S → C → D → G (length 3). Both are optimal (cost 3).

---

### Exercise 2

Prove that a consistent heuristic is admissible.

#### Solution 2

**Given:** For all n, n' (where n' is a successor of n via action a): h(n) ≤ c(n, a, n') + h(n'), and h(goal) = 0.

**To prove:** For all n: h(n) ≤ h*(n).

Let G be the nearest goal from n, and let n = n_0 → n_1 → ... → n_k = G be an optimal path of cost h*(n).

Apply the consistency condition along each edge:

```
h(n_0) ≤ c(n_0, a_1, n_1) + h(n_1)
h(n_1) ≤ c(n_1, a_2, n_2) + h(n_2)
...
h(n_{k-1}) ≤ c(n_{k-1}, a_k, n_k) + h(n_k)
```

Substituting recursively:

```math
h(n_0) \leq \sum_{i=1}^{k} c(n_{i-1}, a_i, n_i) + h(n_k) = h^*(n_0) + 0 = h^*(n_0)
```

Therefore h is admissible. QED.

---

### Exercise 3

For the 8-puzzle, which of the following is an admissible heuristic? Explain.

(a) h(n) = number of tiles not in their goal row  
(b) h(n) = number of tiles in their goal position  
(c) h(n) = Manhattan distance of the blank tile to its goal position  

#### Solution 3

**(a) Number of tiles not in their goal row.** Admissible. Each tile not in its correct row requires at least one vertical move, so this counts a lower bound on required moves. The heuristic relaxes the problem by ignoring columns and ignoring that only one tile can move at a time.

**(b) Number of tiles in their goal position.** NOT admissible as an *estimate of cost*. This counts tiles that are already solved — it decreases as the puzzle approaches the goal, making it a measure of progress, not a cost estimate. It equals (8 - misplaced_tiles), which can be greater than h*(n) in many states. For example, in the goal state h = 8 but h*(n) = 0 — a clear overestimate if we used it as a cost.

**(c) Manhattan distance of the blank tile.** Admissible but very weak. Moving the blank one step requires exactly one move, but the blank is not a numbered tile — its goal position matters only to create space. Using only the blank's distance gives h(n) ≤ 1 for most states (blank is usually adjacent to its goal), which is a valid lower bound but nearly useless. Use Manhattan distance over *all tiles* (h_2) instead.

---

## Sources

- Russell, S. & Norvig, P. *Artificial Intelligence: A Modern Approach*, 4th ed. Chapters 3–4. Pearson, 2020.
- Lecture slides: air(19).pdf — Search Agents; air(20).pdf — Uninformed Search; air(21).pdf — Informed Search (University course materials, 2025).
- Nilsson, N. *Problem-Solving Methods in Artificial Intelligence*. McGraw-Hill, 1971. (Original IDA* motivation.)
- Korf, R. E. "Depth-first iterative-deepening: an optimal admissible tree search." *Artificial Intelligence* 27(1):97–109, 1985.
- Conversation with user, 2026-05-19.

## Related

- [Intelligent Agents](./intelligent-agents.md)
- [Adversarial Search and CSPs](./3-adversarial-search-and-csps.md)
- [Logical Agents and Knowledge Representation](./4-logical-agents-and-knowledge-representation.md)
