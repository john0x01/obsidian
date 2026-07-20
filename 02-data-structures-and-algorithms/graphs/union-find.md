# Union-Find (Disjoint Set)

Union-Find (a.k.a. the disjoint-set data structure) maintains a partition of `n` elements into disjoint sets and supports two operations blisteringly fast: **`find(x)`** returns a canonical representative of `x`'s set, and **`union(x, y)`** merges the two sets. It answers "are these two things connected?" and "connect them" incrementally — the killer feature being that with the right optimisations, both run in **effectively constant amortised time**.

## The Disjoint-Set Forest

Represent each set as a tree; every element points to a **parent**, and the root is the set's representative. `find` walks parent pointers to the root; `union` links one root under the other. Two elements are in the same set iff they share a root.

```ts
class UnionFind {
  parent: number[]; rank: number[];
  constructor(n: number) { this.parent = [...Array(n).keys()]; this.rank = new Array(n).fill(0); }
  find(x: number): number {
    while (this.parent[x] !== x) { this.parent[x] = this.parent[this.parent[x]]; x = this.parent[x]; }
    return x;                       // path halving: point each node to its grandparent
  }
  union(x: number, y: number): boolean {
    let rx = this.find(x), ry = this.find(y);
    if (rx === ry) return false;    // already connected — no merge, signals a cycle
    if (this.rank[rx] < this.rank[ry]) [rx, ry] = [ry, rx];
    this.parent[ry] = rx;
    if (this.rank[rx] === this.rank[ry]) this.rank[rx]++;
    return true;
  }
}
```

Naïvely, trees can degenerate into linked lists, making `find` `O(n)`. Two optimisations fix this, and their *combination* is what produces the famous bound.

## The Two Optimisations — and the α(n) Bound

**Union by rank (or size).** Always attach the shorter tree under the taller one (rank ≈ upper bound on height). This alone keeps tree height `O(log n)`, so each operation is `O(log n)`.

**Path compression.** During `find`, flatten the path so visited nodes point closer to (or directly at) the root. Repeated `find`s on the same chain amortise the flattening cost. Alone, path compression also gives `O(log n)` amortised.

**Together**, the amortised cost per operation drops to **`O(α(n))`**, where `α` is the *inverse Ackermann function* — it grows so slowly that `α(n) ≤ 4` for any `n` that could exist in the physical universe (Ackermann's function explodes past `2^65536` by its fourth value). So in practice each operation is **constant time**, though it is *not* truly `O(1)` — the bound is amortised and the constant is inverse-Ackermann, a subtlety worth stating precisely (see [[amortized-analysis]]). This is one of the most elegant results in data structures: two one-line tricks collapse `O(log n)` to a constant for all conceivable inputs.

Note the two flavours: **union by rank** tracks tree depth; **union by size** tracks element count. Both achieve the same asymptotic bound; size is handier when you also need each component's cardinality.

## Applications

- **[[minimum-spanning-tree|Kruskal's MST]].** The dominating use: as edges are considered in weight order, `union` returns `false` exactly when both endpoints already share a set — i.e. adding the edge would close a cycle. This is the cycle test that makes Kruskal near-linear after the sort.
- **Dynamic connectivity.** Answer a stream of "connect `a`–`b`" and "are `a`, `b` connected?" queries online, in near-constant time each — something [[graph-traversal|BFS/DFS]] cannot do without re-traversing per query. (Union-Find is *incremental*: it handles unions and queries but **not deletions** — edge removal needs heavier machinery.)
- **Cycle detection in undirected graphs.** Process edges; if `union(u, v)` finds `u` and `v` already joined, you've found a cycle. `O(E · α(V))` overall.
- **Connected components / percolation / image labeling / equivalence-class merging** (e.g. grouping `==` constraints), and grid flood-grouping problems.

## Senior Pitfalls

- **Using only one optimisation.** Path compression *or* union-by-rank gives `O(log n)`; you need *both* for the α(n) bound. Interviewers probe this.
- **Comparing `find(x) == find(y)` naïvely for "connected"** is correct — but comparing raw `parent[x] == parent[y]` is a bug; only roots are canonical.
- **Recursive `find` with compression** can overflow the stack on a pre-compression degenerate chain of millions; the iterative path-halving above sidesteps that (see [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]]).
- **No efficient split/delete.** If the problem removes edges over time (fully dynamic connectivity), plain Union-Find is the wrong tool.
- The α(n) figure is **amortised, not per-operation worst case** — a single early `find` before compression can still be `O(log n)`. Fine for batch algorithms, worth flagging in latency-sensitive contexts.

## See Also
- [[minimum-spanning-tree]] — Kruskal's algorithm's core cycle test
- [[amortized-analysis]] — where the inverse-Ackermann bound comes from
- [[binary-trees]] — the disjoint-set forest as a (flattened) tree structure
- [[graph-traversal]] — the alternative for one-shot (non-incremental) connectivity
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]] — why "α(n) ≈ constant" holds in the real world
