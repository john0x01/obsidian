# Shortest Paths

The shortest-path problem asks for the minimum-total-weight route between vertices. There is no single algorithm — the right one depends entirely on **what the edge weights look like** (absent, non-negative, possibly negative) and **how many source/target pairs** you need. Picking the wrong one is either needlessly slow or silently wrong, so the discriminators below matter more than the code.

## Unweighted: Just Use BFS

If edges are unweighted (or uniform), [[graph-traversal|BFS]] from the source computes shortest distances in **`O(V + E)`**, because it visits vertices in nondecreasing hop-distance. Don't reach for anything heavier. (A generalisation, **0–1 BFS** with a deque, handles weights that are only 0 or 1 in the same `O(V+E)`.)

## Dijkstra — Non-Negative Weights, Greedy + Heap

Dijkstra maintains a set of vertices with **finalised** shortest distances and repeatedly extracts the unfinalised vertex with the smallest tentative distance, then **relaxes** its outgoing edges. The [[priority-queues|priority queue]] (a binary [[heaps|heap]]) is what makes "extract the current minimum" cheap.

```ts
function dijkstra(adj: [number, number][][], src: number): number[] {
  const dist = new Array(adj.length).fill(Infinity); dist[src] = 0;
  const pq: [number, number][] = [[0, src]];    // [dist, vertex] min-heap
  while (pq.length) {
    const [d, u] = heapPop(pq);
    if (d > dist[u]) continue;                   // stale entry, skip
    for (const [v, w] of adj[u])
      if (d + w < dist[v]) { dist[v] = d + w; heapPush(pq, [dist[v], v]); }
  }
  return dist;
}
```

With a binary heap this is **`O((V + E) log V)`** — every edge can trigger one heap push (`E log V`) and every vertex one extract (`V log V`). A Fibonacci heap lowers it to `O(E + V log V)` in theory, but the constants make it rarely worth it in practice (see [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]).

**Why negative edges break it.** Dijkstra's correctness rests on the greedy invariant that *once a vertex is extracted, its distance is final*. That holds only if no future path can be cheaper — true precisely because all remaining edges are non-negative, so extending any path can only add cost. A **negative edge** violates this: a vertex could be finalised at distance 5, then a later-discovered negative edge could have reached it at 2 — but Dijkstra never revisits it. It doesn't loop or crash; it just returns wrong answers. This is a favourite senior interview trap.

## Bellman–Ford — Negatives & Cycle Detection

When weights can be negative, relax **every edge, `V − 1` times**. After `k` rounds, all shortest paths using at most `k` edges are correct; since a simple shortest path has ≤ `V − 1` edges, `V − 1` rounds suffice.

```ts
for (let i = 0; i < V - 1; i++)
  for (const [u, v, w] of edges)
    if (dist[u] + w < dist[v]) dist[v] = dist[u] + w;
// one more pass: any further relaxation ⟹ a negative cycle is reachable
```

**`O(V·E)`** — slower than Dijkstra, but it does two things Dijkstra can't: it tolerates negative edges, and a *`V`-th* relaxation pass that still improves some distance **proves a negative cycle exists** (shortest paths become undefined — you can loop forever getting cheaper). This cycle-detection ability is used in currency-arbitrage detection and constraint systems.

## Floyd–Warshall — All-Pairs DP

For **all** `V²` source–target pairs on a small/dense graph, Floyd–Warshall is a compact triple loop: `dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])`, iterating the intermediate vertex `k` in the *outer* loop.

```ts
for (let k = 0; k < V; k++)
  for (let i = 0; i < V; i++)
    for (let j = 0; j < V; j++)
      if (dist[i][k] + dist[k][j] < dist[i][j]) dist[i][j] = dist[i][k] + dist[k][j];
```

The `k`-outer ordering is the [[dynamic-programming|DP]] invariant: after iteration `k`, `dist[i][j]` is optimal using only intermediates `{0..k}`. **`O(V³)` time, `O(V²)` space.** Handles negative edges (a negative diagonal entry flags a negative cycle) but not competitively for single-source on large sparse graphs — run `V` Dijkstras instead.

## A\* — Heuristic-Guided Search

A\* is Dijkstra biased toward the goal by a **heuristic** `h(v)` estimating remaining cost, prioritising on `f(v) = g(v) + h(v)`. It expands far fewer nodes when the heuristic is informative (Euclidean distance for maps, Manhattan for grids).

- **Admissible** (`h` never overestimates the true remaining cost) ⟹ A\* returns an **optimal** path.
- **Consistent/monotone** (`h(u) ≤ w(u,v) + h(v)`) ⟹ no node is ever reopened; A\* behaves like Dijkstra on reduced edge costs. Consistency implies admissibility. With `h ≡ 0`, A\* *is* Dijkstra.

## Choosing

| Situation | Use |
|---|---|
| Unweighted | BFS, `O(V+E)` |
| Non-negative weights, single source | Dijkstra, `O((V+E) log V)` |
| Negative edges / need cycle detection | Bellman–Ford, `O(VE)` |
| All pairs, small dense graph | Floyd–Warshall, `O(V³)` |
| Single target + good heuristic | A\* |

**Pitfalls:** applying Dijkstra with negative edges (wrong, silently); forgetting the stale-entry skip (`d > dist[u]`) in a heap-based Dijkstra, which still works but wastes time; assuming a *unique* shortest path; and integer **overflow** when summing large weights toward `Infinity`/`INT_MAX` sentinels.

## See Also
- [[priority-queues]] — the min-priority queue at Dijkstra's and A\*'s core
- [[heaps]] — the binary heap giving `O(log V)` extract-min
- [[dynamic-programming]] — Bellman–Ford and Floyd–Warshall as DP recurrences
- [[greedy-algorithms]] — Dijkstra's greedy finalisation and its non-negativity precondition
- [[graph-traversal]] — BFS as the unweighted shortest-path base case
