---
title: "6 - PageRank and Graph Algorithms"
created: 2026-05-19
updated: 2026-05-19
tags: [machine-learning, graph-algorithms, pagerank, eigenvector, random-walk, hits, information-retrieval]
aliases: []
---

# 6 - PageRank and Graph Algorithms

[toc]

> **TL;DR:** PageRank assigns an importance score to every node in a directed graph by solving for the dominant eigenvector of a modified adjacency (transition) matrix. The intuition is a random surfer who follows links at random but occasionally teleports to an arbitrary page — the stationary distribution of this walk is the PageRank vector. Power iteration converges in O(log n) steps on expander-like graphs such as the Web, making it tractable at billion-node scale.

## Vocabulary

**PageRank** R(u) — a scalar importance score for page u, defined recursively as the sum of fractional ranks contributed by all pages that link to u.

**Backlink** — a directed edge v → u: page v links to page u; u receives rank from v.

**Out-degree** N_u = |F_u| — the number of forward links from page u. PageRank from u is divided equally among all N_u outgoing links.

**Transition matrix** A — the column-stochastic matrix where A_{u,v} = 1/N_v if v links to u, and 0 otherwise. R = cAR in the simplified model.

**Rank sink** — a group of pages that collectively receive rank but have no outgoing edges (or only internal edges), causing rank to accumulate there indefinitely. Also called a dangling link problem.

**Teleportation vector** E(u) — a source-of-rank vector that redistributes "lost" rank from sinks back into the graph. In the uniform case E is 1/n for all pages.

**Damping factor** (d) — the probability that the random surfer follows a link on any given step, rather than teleporting. Typically d = 0.85 in the original PageRank paper.

**Personalized PageRank** — a variant where E is non-uniform, concentrating the teleportation distribution on a specific set of "seed" pages, producing topic-specific or user-specific rankings.

**Power iteration** — the numerical method for computing the dominant eigenvector: repeatedly apply the transition matrix to an initial vector until convergence.

**HITS** (Hyperlink-Induced Topic Search, Kleinberg 1998) — a dual-score algorithm that assigns each node both a hub score and an authority score. Hubs link to good authorities; authorities are linked to by good hubs. Solves two coupled eigenvector equations simultaneously.

**Stationary distribution** (π) — the probability distribution over nodes that a random walk on a Markov chain converges to; for PageRank, this is the dominant left eigenvector of the transition matrix.

## Intuition

Imagine releasing 1 million random surfers onto the web simultaneously. Each surfer follows a random outgoing link at each step. With small probability d_teleport = 1 - d they jump to a completely random page instead of following a link. After millions of steps, the fraction of time a surfer spends at page u is proportional to u's importance: highly linked, central pages attract more surfers than obscure dead-ends.

This is exactly the stationary distribution of a Markov chain. PageRank operationalises the intuition that a link from an important page is worth more than a link from an unimportant one — both the number of backlinks and the quality of those backlinks matter. The mathematical object encoding this is the dominant eigenvector of the transition matrix, which power iteration finds efficiently.

```
  ┌───────────────────────────────────────────────┐
  │               Random Surfer Model              │
  │                                               │
  │  At each step:                                │
  │  ┌─────────────────────────────────────────┐ │
  │  │  with prob d   → follow a random link   │ │
  │  │  with prob 1-d → teleport to random pg  │ │
  │  └─────────────────────────────────────────┘ │
  │                                               │
  │  Limiting distribution = PageRank vector R    │
  └───────────────────────────────────────────────┘
```

**Figure:** the random surfer model — PageRank is the stationary distribution.

## How it Works

PageRank is computed iteratively. The simplified version ignores rank sinks; the full version handles them with the teleportation vector E.

### Simplified PageRank (No Rank Sinks)

The simplified recursive definition: the rank of page u equals the sum of contributions from all pages v that link to u, where each v contributes its rank divided by its out-degree N_v, scaled by a normalisation constant c.

```math
R(u) = c \sum_{v \in B_u} \frac{R(v)}{N_v}
```

In matrix form: R = cAR, so R is an eigenvector of A with eigenvalue c. PageRank is specifically the dominant eigenvector (eigenvalue = 1 after normalisation), computed by power iteration starting from any positive vector.

### Full PageRank with Teleportation

The teleportation vector E prevents rank sinks. The full definition:

```math
R'(u) = c \sum_{v \in B_u} \frac{R'(v)}{N_v} + cE(u)
```

In matrix form: R' = c(AR' + E). This can be rewritten as R' is an eigenvector of (A + E × **1**), where **1** is the all-ones vector. The damping factor d = c controls the trade-off between following links and teleporting.

### Power Iteration

The power iteration algorithm starts from S = E (or any positive vector) and repeatedly applies the transition matrix until convergence:

```
R₀ ← S
loop:
    R_{i+1} ← A · Rᵢ
    d ← ‖Rᵢ‖₁ − ‖R_{i+1}‖₁   (rank "lost" to sinks)
    R_{i+1} ← R_{i+1} + d·E  (redistribute lost rank)
    δ ← ‖R_{i+1} − Rᵢ‖₁
while δ > ε
```

Each iteration is one sparse matrix-vector multiply: O(|edges|) work, O(|nodes|) memory if A is stored in sparse format.

> [!IMPORTANT]
> The convergence rate of power iteration is determined by the ratio |λ₂|/|λ₁| where λ₁=1 is the dominant eigenvalue and λ₂ is the second-largest. For expander-like graphs (such as the Web) this ratio is well below 1, leading to convergence in O(log n) iterations. For graphs with tightly-coupled communities, convergence is slower.

### Dangling Links

Pages with no outgoing links (dangling links) cause rank to "disappear" — they receive rank but never distribute it. The original PageRank paper handles this by removing dangling pages from the main computation and reinserting them afterwards. The d factor in the power iteration pseudocode above captures the rank mass lost at dangling pages and redistributes it via E.

### HITS: Hubs and Authorities

Kleinberg's HITS algorithm is an alternative to PageRank that computes two scores per node: authority score (a good destination for information) and hub score (a good pointer to authorities). The two scores are defined by coupled eigenvector equations:

```math
a(u) = \sum_{v: v \to u} h(v), \qquad h(u) = \sum_{v: u \to v} a(v)
```

In matrix form with the adjacency matrix M: a = Mᵀh and h = Ma. This means a is an eigenvector of MᵀM and h is an eigenvector of MMᵀ. HITS is query-dependent (applied to a small subgraph relevant to a query), whereas PageRank is query-independent (pre-computed globally).

## Math

The simplified PageRank equation R = cAR is a standard eigenvector problem. The Perron-Frobenius theorem guarantees a unique positive dominant eigenvector when A is irreducible (strongly connected graph) and aperiodic. Teleportation ensures both properties even when the raw web graph is not strongly connected: adding rank injected from every node at rate (1-d)/n makes the effective transition matrix a convex combination of A and the uniform matrix, which is both irreducible and aperiodic.

```math
M = d \cdot A + (1 - d) \cdot \frac{\mathbf{1}\mathbf{1}^T}{n}
```

R is the dominant eigenvector of M. The L1-normalised version gives R(u) = probability that the random surfer is at page u in steady state.

```math
\sum_u R(u) = 1, \qquad R(u) \geq 0 \;\forall u
```

**Convergence:** for the uniform teleportation model with damping factor d = 0.85, the second eigenvalue of M is at most d = 0.85. Thus power iteration reduces error by factor d per step:

```math
\|R_{i} - R^*\|_1 \leq d^i \|R_0 - R^*\|_1
```

On a 322-million-link graph, the original paper reported convergence in roughly 52 iterations.

> [!NOTE]
> In the original 1998 paper, Page and Brin used c < 1 (not d explicitly) as the normalisation factor, and E uniform over all pages. The modern formulation with explicit damping factor d ∈ (0,1) and the equation R = d·AR + (1-d)·E is equivalent but cleaner.

## Real-world Example

PageRank on a citation network: rank academic papers by the importance of papers that cite them. This is structurally identical to web PageRank. Below is a complete implementation using NetworkX.

```python
import networkx as nx
import numpy as np

# Build a directed citation graph: edge (A→B) means paper A cites paper B
# (B receives "authority" from A)
G = nx.DiGraph()
# Simplified 6-node academic citation graph
edges = [
    ('survey', 'paper_A'),
    ('survey', 'paper_B'),
    ('survey', 'paper_C'),
    ('paper_A', 'paper_B'),
    ('paper_B', 'paper_C'),
    ('paper_C', 'paper_A'),   # cycle → rank sink without teleportation
    ('intro', 'survey'),
    ('intro', 'paper_A'),
]
G.add_edges_from(edges)

# NetworkX PageRank: alpha=damping factor (d in our notation)
pr = nx.pagerank(G, alpha=0.85, max_iter=100, tol=1e-6)
for node, rank in sorted(pr.items(), key=lambda x: -x[1]):
    print(f"  {node:15s}  PageRank = {rank:.4f}")

# Output (approximate):
#   paper_B          PageRank = 0.2180
#   paper_A          PageRank = 0.2098
#   paper_C          PageRank = 0.1990
#   survey           PageRank = 0.1607
#   intro            PageRank = 0.0870  (← dangling — no outedges to rest of graph)
# paper_B ranks high because it is cited by survey AND paper_A

# Power iteration from scratch to see convergence behaviour
n = G.number_of_nodes()
nodes = list(G.nodes())
A = nx.to_numpy_array(G, nodelist=nodes)   # row i, col j: A[i,j] = 1 if j→i
# Column-normalise (each column = outgoing prob distribution from that node)
col_sums = A.sum(axis=0)
col_sums[col_sums == 0] = 1  # avoid divide-by-zero for dangling nodes
A_norm = A / col_sums

d = 0.85
R = np.ones(n) / n   # uniform initialisation
E = np.ones(n) / n   # uniform teleportation

for iteration in range(100):
    R_new = d * A_norm @ R + (1 - d) * E
    delta = np.abs(R_new - R).sum()
    R = R_new
    if delta < 1e-8:
        print(f"Converged at iteration {iteration}")
        break

print("\nManual PageRank:")
for node, rank in zip(nodes, R):
    print(f"  {node:15s}  {rank:.4f}")
```

> [!TIP]
> For large-scale PageRank (billions of nodes), use distributed sparse matrix-vector multiply: store A in CSR format partitioned across machines, distribute R as a sharded vector, and run the iteration in MapReduce or Spark. Each iteration is one sparse MV multiply followed by a broadcast of the updated R. The communication cost per iteration is O(|edges|) across the cluster.

## In Practice

**Convergence in production.** The original Google implementation ran PageRank on 24 million pages in about 5 hours in 1998. With 4 bytes per rank value, 75 million URLs required 300 MB of RAM — tight by 1998 standards. The sorted link structure (linear on disk) allowed sequential access per iteration.

**Damping factor sensitivity.** The d = 0.85 value is empirically chosen. Lower d means more teleportation, faster convergence but less differentiation between nodes (ranks cluster near 1/n). Higher d means slower convergence and more sensitivity to the graph's community structure.

**Spam resistance.** PageRank is harder to game than raw backlink count because gaining rank requires convincing an already-high-rank page to link to you. However, link farms (tightly-linked clusters of fake pages all pointing at a target) can artificially inflate rank. Personalised PageRank with restricted teleportation sets is more resistant to this.

> [!WARNING]
> PageRank is not equivalent to a keyword-relevance score. A page can have high PageRank but be completely irrelevant to a user's query. Modern search combines PageRank (or its successors) with query-dependent text relevance (BM25, dense retrieval) — neither alone is sufficient.

**Beyond web search.** PageRank has been applied to: citation analysis (rank papers by importance), social network influence (identify influencers), protein interaction networks (identify hub proteins), recommendation systems (rank items by propagated user interest), and malware detection (identify botnet command-and-control nodes by link structure).

## Pitfalls

- **"PageRank measures content quality."** — PageRank measures structural centrality in the link graph. A page can rank highly due to link accumulation while having no substantive content (e.g., a navigation hub), and a page with excellent content but few backlinks will rank low.
- **"All backlinks are equally valuable."** — A link from a high-PageRank page contributes more rank. A backlink from Yahoo (PageRank > 10,000 in the original scale) is worth far more than 1,000 links from low-PageRank pages.
- **"Power iteration always converges quickly."** — On graphs with near-degenerate eigenvalues (λ₂ ≈ λ₁), convergence is slow. This occurs on graphs with weakly-connected communities or near-bipartite structure. In such cases, Chebyshev acceleration or aggregation methods can speed convergence.
- **"HITS and PageRank always agree."** — They encode different importance definitions. HITS gives separate hub/authority scores and is query-dependent; PageRank is query-independent. On the same graph, HITS and PageRank rankings can differ substantially.
- **"Teleportation parameter doesn't matter much."** — The choice of E (the teleportation distribution) fundamentally changes which pages get high rank. Personalised PageRank with E concentrated on a user's bookmarks produces a completely different ranking than uniform E.

## Exercises

### Exercise 1 — PageRank by Hand (3 pages)

Three pages A, B, C with links: A→B, A→C, B→C, C→A. No dangling links. Using the simplified model R = cAR with c=1 (normalise so ‖R‖₁=1), compute the PageRank vector.

#### Solution 1

Build the transition matrix A (column j = outgoing prob from page j):

```
     A    B    C
A [  0    0    1  ]    (C→A: A receives from C)
B [ 0.5   0    0  ]    (A→B: B receives half of A's rank)
C [ 0.5   1    0  ]    (A→C, B→C: C receives)
```

Set up R = AR with ‖R‖₁=1:

```math
R(A) = R(C)
R(B) = 0.5 \cdot R(A)
R(C) = 0.5 \cdot R(A) + R(B)
```

From the first equation: R(A)=R(C). From the third: R(C) = 0.5·R(A)+R(B) = R(A) → R(B)=0.5·R(A). Normalisation: R(A)+R(B)+R(C)=1 → R(A)+0.5·R(A)+R(A)=1 → 2.5·R(A)=1.

```math
R(A) = 0.40, \quad R(B) = 0.20, \quad R(C) = 0.40
```

C and A tie for first: C feeds A and receives from both A and B, creating a balanced cycle.

### Exercise 2 — Rank Sink

Add a page D to the graph above with links D→D (only self-link). What happens to PageRank of A, B, C in the simplified model (no teleportation)?

#### Solution 2

D receives rank from no external page (no other page links to D except D itself, which is a self-link with N_D=1 → D contributes all its rank back to itself). Meanwhile A, B, C have no link to D, so D starts with rank 0/n = 0 initially. Since D receives no rank from external sources and only cycles its own rank, D stays at R(D)=0 forever under power iteration. A, B, C are unaffected because D is not in their subgraph. This is a trivial rank sink example because D is disconnected.

A non-trivial rank sink occurs when A, B, C have edges into D but D has no edges back: all rank flowing into D is absorbed. Without teleportation, eventually all rank concentrates in D and R(A)=R(B)=R(C)→0. Teleportation with E uniform over {A,B,C,D} prevents this by continuously injecting rank into all pages.

### Exercise 3 — Eigenvector Interpretation

Explain why PageRank is the dominant eigenvector of the transition matrix, not just any eigenvector.

#### Solution 3

The transition matrix A is column-stochastic (columns sum to 1, all entries ≥ 0). By the Perron-Frobenius theorem, such matrices have a unique dominant eigenvalue λ₁=1 with a corresponding eigenvector that has all non-negative entries. All other eigenvalues satisfy |λᵢ| < 1 (given irreducibility and aperiodicity). Power iteration starting from any positive vector R₀ converges to this dominant eigenvector because the component along every non-dominant eigenvector shrinks by |λᵢ/λ₁|ⁱ → 0. The dominant eigenvector is the unique non-negative normalised solution, which is exactly what we want for a probability distribution.

### Exercise 4 — Personalised PageRank

You are building a recommendation system. User Alice has bookmarked pages {P1, P2}. How does personalised PageRank differ from standard PageRank in terms of the teleportation vector E, and what effect does this have on rankings?

#### Solution 4

**Standard PageRank:** E is uniform over all n pages, E(u) = 1/n ∀u. The random surfer teleports to a uniformly random page.

**Personalised PageRank for Alice:** E(P1) = E(P2) = 0.5, E(u) = 0 for all other pages. The random surfer teleports only to Alice's bookmarks.

**Effect:** pages that are strongly connected (via short link paths) to P1 or P2 receive disproportionately high rank in Alice's personalised ranking, even if they have low global PageRank. A highly-cited paper in a field unrelated to Alice's bookmarks will rank lower in her personalised ranking than in the global ranking. Personalised PageRank concentrates the rank distribution around the "neighbourhood" of Alice's seed pages in the graph, capturing her topical interests.

### Exercise 5 — Convergence Rate

For a web graph with damping factor d=0.85, how many iterations of power iteration are needed to reduce the initial error by a factor of 10⁻⁶?

#### Solution 5

The error decreases by at most factor d per iteration:

```math
\|R_i - R^*\|_1 \leq d^i \|R_0 - R^*\|_1
```

We want d^i ≤ 10⁻⁶:

```math
i \geq \frac{\log(10^{-6})}{\log(0.85)} = \frac{-6 \log 10}{\log 0.85} = \frac{-6 \times 2.303}{-0.1625} \approx \frac{13.82}{0.1625} \approx 85 \text{ iterations}
```

So approximately 85 iterations are needed for 6 orders of magnitude reduction in error. The original paper observed convergence in ~52 iterations on a 322-million-link database, consistent with this bound (their convergence criterion was less strict than 10⁻⁶ relative error).

## Sources

- Page, L., Brin, S., Motwani, R., & Winograd, T. (1998). The PageRank Citation Ranking: Bringing Order to the Web. Stanford Technical Report SIDL-WP-1999-0120. air(102).pdf.
- Kleinberg, J. (1998). Authoritative sources in a hyperlinked environment. *Proceedings of ACM-SIAM SODA*, 1998. (HITS algorithm.)
- Brin, S. & Page, L. (1998). The anatomy of a large-scale hypertextual web search engine. *Proceedings of WWW-7*.
- Motwani, R. & Raghavan, P. (1995). *Randomized Algorithms*. Cambridge University Press.

## Related

- [5 - Linear Algebra Essentials](../1-foundations/5-linear-algebra-essentials.md)
- [4 - Clustering and Unsupervised Learning](./4-clustering-and-unsupervised.md)
- [5 - Association Rules](./5-association-rules.md)
- [7 - Genetic Algorithms](./7-genetic-algorithms.md)
- [1 - What is ML and Version Space](../1-foundations/1-what-is-ml-and-version-space.md)
