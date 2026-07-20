# Graphs

A graph `G = (V, E)` is a set of **vertices** connected by **edges** — the most general relational data structure, modelling anything expressible as "things and the connections between them": road networks, social follows, package dependencies, state machines, the web. Trees, linked lists, and grids are all special-case graphs, so the graph vocabulary and algorithms subsume much of the rest of this track.

## Terminology You Must Have Cold

- **Vertex (node)** and **edge (arc)**. An edge may carry a **weight** (cost, distance, capacity).
- **Directed vs undirected.** In a directed graph an edge `u→v` is one-way; undirected edges are symmetric. Directedness changes everything downstream — reachability, connectivity, and cycle definitions all differ.
- **Degree.** For undirected graphs, the number of incident edges; for directed, split into **in-degree** and **out-degree**. The handshake lemma: `Σ deg(v) = 2E`.
- **Path / cycle.** A path is a sequence of distinct vertices joined by edges; a **cycle** returns to its start. A graph with no cycles is **acyclic**. A directed acyclic graph is a **DAG** — the substrate for [[topological-sort]] and dependency resolution.
- **Connected.** An undirected graph is connected if every vertex is reachable from every other; the maximal connected pieces are **connected components**. The directed analogue is **strong connectivity** (see [[strongly-connected-components]]).
- **Dense vs sparse.** `E` ranges from `O(V)` (sparse — trees, road networks, most real graphs) up to `O(V²)` (dense — near-complete graphs). This ratio drives representation and algorithm choice more than any other single fact.

## Representations — the Decision That Sets Your Complexity

Everything an algorithm can do cheaply is dictated by how you store the graph. Three canonical choices:

**Adjacency matrix.** A `V×V` boolean/weight matrix, `M[u][v]` = edge weight or 0/∞.
```ts
const M: number[][] = Array.from({length: V}, () => new Array(V).fill(0));
M[u][v] = w;                 // add edge
const connected = M[u][v] !== 0;   // O(1) edge test
```
- **Space `O(V²)`** regardless of edge count — wasteful for sparse graphs.
- **`O(1)` edge existence/weight lookup and update** — its one superpower.
- But iterating a vertex's neighbours is **`O(V)`** (you scan a whole row), and a full traversal is **`O(V²)`**.
- Use when the graph is **dense**, small, or when you constantly ask "is there an edge `u→v`?" (e.g. [[shortest-paths|Floyd–Warshall]], which is inherently `O(V³)` anyway).

**Adjacency list.** Each vertex stores a list of its neighbours (with weights).
```ts
const adj: Array<Array<[number, number]>> = Array.from({length: V}, () => []);
adj[u].push([v, w]);         // O(1) add edge
for (const [nbr, w] of adj[u]) { /* iterate only real edges */ }
```
- **Space `O(V + E)`** — proportional to what actually exists.
- Neighbour iteration is **`O(deg(v))`**, and a full traversal is **`O(V + E)`** — optimal.
- Edge existence test is **`O(deg(v))`** unless you back each list with a hash set (then `O(1)` expected, at the cost of memory).
- The **default for almost every real graph and every traversal-based algorithm** ([[graph-traversal]], Dijkstra, Prim).

**Edge list.** Just an array of `(u, v, w)` triples. `O(E)` space, `O(E)` to answer anything vertex-centric, but ideal when an algorithm processes **all edges as a batch** — Kruskal's [[minimum-spanning-tree]] (which sorts edges) and Bellman–Ford (which relaxes every edge each round) both want this form.

Rule of thumb: **adjacency list by default**; matrix only when dense or edge-lookup-dominated; edge list when the algorithm iterates edges globally.

## What Runs on Graphs

The rest of this cluster is essentially a catalogue of graph algorithms, each tuned to a structural assumption:

- **Traversal** — [[graph-traversal|BFS and DFS]], `O(V+E)`, the foundation for reachability, components, cycle detection, and edge classification.
- **Ordering** — [[topological-sort]] on DAGs.
- **Shortest paths** — [[shortest-paths]]: BFS (unweighted), Dijkstra (non-negative weights), Bellman–Ford (negatives + cycle detection), Floyd–Warshall (all-pairs), A\* (heuristic).
- **Connectivity structure** — [[union-find]] for incremental undirected connectivity, [[strongly-connected-components]] for directed.
- **Spanning structure** — [[minimum-spanning-tree]] via Kruskal/Prim.

**Senior pitfalls.** Watch the `O(V²)` matrix memory blow-up on large sparse graphs — a million-vertex social graph as a matrix is a terabyte. Beware silently allowing **self-loops or parallel edges** if your representation and algorithm don't. In directed graphs, remember to add edges in one direction only — a symmetric bug quietly turns your digraph undirected. And note that a "graph" backing a real system is often on disk or distributed (see [[04-databases/nosql-graph|Graph Databases]]), where the in-memory complexity model breaks down and I/O dominates.

## See Also
- [[graph-traversal]] — BFS/DFS, the base every graph algorithm builds on
- [[04-databases/nosql-graph|Graph Databases]] — graphs as a persistent, queryable store
- [[binary-trees]] — trees are connected acyclic graphs
- [[asymptotic-notation]] — the `O(V+E)` vs `O(V²)` distinction that representation choice hinges on
- [[07-performance-engineering/data-locality|Data Locality]] — why matrices can beat lists despite worse Big-O
