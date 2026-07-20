# Graph Traversal (BFS & DFS)

Traversal is the act of visiting every reachable vertex from a source, exactly once, in some disciplined order. **Breadth-first search (BFS)** and **depth-first search (DFS)** are the two primitives; nearly every other graph algorithm is one of them with bookkeeping bolted on. Both run in **`O(V + E)`** time and **`O(V)`** space on an [[graphs|adjacency list]] — you touch each vertex once and each edge once.

## The Shared Invariant: the Visited Set

The one rule that separates graph traversal from tree traversal is that **graphs have cycles and shared vertices**, so you must remember what you've already seen or you'll loop forever and revisit exponentially. A `visited` set (or color array) enforces the invariant *"each vertex is enqueued/pushed at most once."* That single line is what bounds the work at `O(V + E)` — without it there is no bound at all.

## BFS — Queue, Layers, Shortest Unweighted Paths

BFS expands the frontier level by level using a **FIFO [[queues|queue]]**. Mark a vertex visited *when you enqueue it*, not when you dequeue it, to avoid duplicate enqueues.

```ts
function bfs(adj: number[][], src: number): number[] {
  const dist = new Array(adj.length).fill(-1);
  const q = [src]; dist[src] = 0;
  for (let i = 0; i < q.length; i++) {   // q as growing array = queue
    const u = q[i];
    for (const v of adj[u]) if (dist[v] === -1) { dist[v] = dist[u] + 1; q.push(v); }
  }
  return dist;
}
```

The defining property: because BFS visits vertices in nondecreasing distance from the source, `dist[v]` is the **length of the shortest path in an unweighted graph** (or a graph with unit edge weights). This is the cheapest correct shortest-path algorithm when weights are absent — no priority queue needed (see [[shortest-paths]]). BFS also gives the natural "flood fill" / minimum-number-of-moves structure (grids, puzzles, word ladders).

## DFS — Recursion/Stack, Discovery & Finish Times

DFS plunges as deep as possible before backtracking, via **[[recursion]]** or an explicit **[[stacks|stack]]**. Its power comes from timestamping: record a **discovery time** when you first enter a vertex and a **finish time** when you exit after exploring all descendants.

```ts
function dfs(adj: number[][], u: number, seen: boolean[], disc: number[], fin: number[], t = {v: 0}) {
  seen[u] = true; disc[u] = t.v++;
  for (const v of adj[u]) if (!seen[v]) dfs(adj, v, seen, disc, fin, t);
  fin[u] = t.v++;
}
```

Discovery/finish intervals **nest like balanced parentheses** — this is the parenthesis theorem, and it underpins [[topological-sort]] (reverse finish order) and [[strongly-connected-components|SCC]] algorithms.

### Edge Classification

On a directed graph, the DFS tree partitions every edge `u→v` into exactly four types, decidable from timestamps:
- **Tree edge** — `v` was undiscovered (part of the DFS forest).
- **Back edge** — `v` is an ancestor still on the recursion stack (`disc[v] < disc[u]`, `v` not finished). **A back edge exists iff the directed graph has a cycle** — the canonical directed cycle test.
- **Forward edge** — `v` is a finished descendant.
- **Cross edge** — `v` is in a previously finished subtree.

In an *undirected* graph there are only tree and back edges, and any back edge (to a non-parent) signals a cycle.

## What Traversal Unlocks

- **Connected components** — repeatedly launch a traversal from each unvisited vertex; each launch discovers one component. `O(V + E)` total. (For *incremental* connectivity under edge insertions, prefer [[union-find]].)
- **Reachability / "can I get from A to B"** — a single traversal from A.
- **Bipartite / 2-colouring check** — BFS/DFS assigning alternating colours by layer; if an edge ever connects two same-coloured vertices, the graph has an odd cycle and is **not bipartite**. `O(V + E)`.
- **Cycle detection** — back edge (directed) or three-colour state; the basis for deadlock and dependency-cycle checks.
- Foundations of [[shortest-paths]] (BFS), topological order and SCCs (DFS).

## Senior Pitfalls

- **Recursive DFS blows the call stack** on deep or adversarial graphs (a path of 10⁶ vertices). Convert to an explicit stack for large inputs; the language runtime won't save you (see [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]] — most engines don't do TCO here anyway).
- **Marking visited on dequeue instead of enqueue** in BFS lets a vertex enter the queue many times, degrading toward `O(V·E)` and corrupting distance labels.
- **Directed vs undirected confusion.** A cycle check that works undirected (any back edge) is wrong for digraphs unless you track "on the current recursion stack," not merely "visited."
- BFS finds shortest paths **only when edges are unweighted/uniform** — the instant weights vary, you need Dijkstra. This is the single most common interview mistake.
- The `O(V+E)` bound assumes adjacency lists; on an adjacency **matrix** every traversal is `O(V²)`.

## See Also
- [[tree-traversals]] — traversal without a visited set (trees can't cycle)
- [[queues]] — the FIFO frontier that makes BFS breadth-first
- [[stacks]] — the LIFO structure behind iterative DFS
- [[recursion]] — natural expression of DFS and its call stack
- [[topological-sort]] — DFS finish times, ordered
- [[strongly-connected-components]] — DFS timestamps applied to directed connectivity
