# Minimum Spanning Tree

A **spanning tree** of a connected, undirected, weighted graph is a subset of edges that connects all `V` vertices with exactly `V − 1` edges and no cycle. The **minimum spanning tree (MST)** is the spanning tree of least total edge weight — the cheapest way to wire everything together. It answers "connect all these nodes as cheaply as possible": laying cable, clustering, network design, and approximation subroutines all reduce to it.

## The Cut Property — Why Greedy Works

Both classic algorithms are greedy, and both are justified by a single theorem. A **cut** partitions the vertices into two non-empty sets. The **cut property** states: *for any cut, the minimum-weight edge crossing it belongs to some MST.* (If all weights are distinct, that edge is in the **unique** MST.)

The exchange argument: suppose the minimum crossing edge `e` were absent from an MST `T`. Adding `e` to `T` creates a cycle; that cycle must cross the cut a second time via some edge `f` with `weight(f) ≥ weight(e)`. Swapping `f` for `e` yields a spanning tree no heavier than `T` — so an MST containing `e` exists. Every safe edge either algorithm picks is the light edge of *some* cut, which is exactly why the local greedy choice is globally optimal (contrast with problems where greed fails; see [[greedy-algorithms]]).

## Kruskal — Sort Edges, Union Components

Kruskal is edge-centric: consider edges in increasing weight order and add each one **unless it would form a cycle**. The cycle test is "are these two endpoints already in the same component?" — precisely what [[union-find|union-find]] answers in near-constant time.

```ts
function kruskal(edges: [number, number, number][], V: number): number {
  edges.sort((a, b) => a[2] - b[2]);        // O(E log E)
  const uf = new UnionFind(V);
  let total = 0, used = 0;
  for (const [u, v, w] of edges) {
    if (uf.union(u, v)) { total += w; used++; }  // union returns false if same set
    if (used === V - 1) break;
  }
  return total;
}
```

The **sort dominates: `O(E log E)`**, which equals `O(E log V)` since `E ≤ V²` implies `log E ≤ 2 log V`. The union-find operations add only `O(E · α(V))` — effectively linear. Each accepted edge is the lightest edge crossing the cut between the two components it joins, so the cut property guarantees correctness. Kruskal naturally handles **disconnected graphs** too, producing a minimum spanning *forest*.

## Prim — Grow One Tree from a Seed

Prim is vertex-centric: start from any vertex and repeatedly add the **cheapest edge that connects a new vertex to the growing tree**. The frontier of candidate edges lives in a [[priority-queues|priority queue]] keyed by edge weight — the cut here is (tree vertices) vs (the rest), and the min-priority edge is exactly the light crossing edge.

```ts
function prim(adj: [number, number][][], V: number): number {
  const inTree = new Array(V).fill(false);
  const pq: [number, number][] = [[0, 0]];     // [weight, vertex]
  let total = 0;
  while (pq.length) {
    const [w, u] = heapPop(pq);
    if (inTree[u]) continue;
    inTree[u] = true; total += w;
    for (const [v, wt] of adj[u]) if (!inTree[v]) heapPush(pq, [wt, v]);
  }
  return total;
}
```

With a binary [[heaps|heap]] and adjacency list this is **`O(E log V)`** — each edge may be pushed once (`E log V`), each vertex extracted once. With a Fibonacci heap it improves to `O(E + V log V)`; with a plain array/matrix and no heap it is **`O(V²)`**, which is actually *best* for dense graphs where `E ≈ V²`.

## Kruskal vs Prim — Which and When

- **Sparse graphs (`E ≈ V`): Kruskal.** Sorting few edges is cheap and the union-find overhead is negligible. Also the go-to when edges are already sorted or arrive as a stream, and when you may have a disconnected graph (forest).
- **Dense graphs (`E ≈ V²`): Prim**, especially the `O(V²)` array version, which avoids the `O(E log E)` sort and beats heap-based variants when `E` is huge.
- Both give the same MST weight; on distinct weights the MST itself is unique, so they agree edge-for-edge.

## Senior Pitfalls

- **MST ≠ shortest-path tree.** The MST minimises *total* edge weight; it does **not** give shortest paths between vertices. Using an MST as a routing tree is a classic conceptual error — that's [[shortest-paths|Dijkstra]]'s job.
- **Non-unique MSTs** when weights tie. Don't assert a canonical edge set; only the total weight is invariant across ties.
- **Directed graphs don't have MSTs** in this sense — the analogous problem (minimum arborescence) needs Chu–Liu/Edmonds, not Kruskal/Prim.
- Kruskal is only as good as its **union-find**: without [[union-find|path compression and union by rank]] the cycle test degrades to `O(log V)` or worse per edge, and a naïve `O(V)` `find` makes the whole thing `O(EV)`.
- Forgetting the `inTree`/stale-entry skip in Prim's heap loop leaves obsolete entries that must be discarded on pop, or the tree weight is wrong.

## See Also
- [[union-find]] — the disjoint-set structure that makes Kruskal near-linear
- [[priority-queues]] — Prim's frontier of candidate crossing edges
- [[greedy-algorithms]] — the cut property as a proof that local greed is globally optimal
- [[heaps]] — the binary heap behind Prim's `O(E log V)`
- [[shortest-paths]] — contrast: minimum *total* weight vs minimum *path* weight
