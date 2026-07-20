# Strongly Connected Components

A **strongly connected component (SCC)** of a directed graph is a maximal set of vertices in which every vertex is reachable from every other — you can get from any node in the SCC to any other and back. SCCs partition the vertices of any digraph, and contracting each SCC to a single node yields the **condensation**, which is always a [[graphs|DAG]]. SCC decomposition is the directed-graph analogue of connected components, and it's the standard tool for detecting cycles and mutual dependencies in directed systems. Both classic algorithms run in **`O(V + E)`**.

## Why It Matters: the Condensation DAG

Any directed graph collapses onto a DAG of its SCCs: shrink each strongly connected blob to one super-node and keep the edges between blobs. Because SCCs absorb every directed cycle, the result is acyclic — so you can [[topological-sort|topologically sort]] the condensation and reason about the graph's coarse structure. A vertex sits in a **nontrivial SCC** (size > 1, or a self-loop) iff it lies on a directed cycle — this is the precise directed-cycle membership test, stronger than the yes/no back-edge check from plain [[graph-traversal|DFS]].

## Tarjan — One DFS with Low-Link

Tarjan finds all SCCs in a **single DFS pass**. Each vertex gets a `disc` (discovery index) and a `low` value — the smallest discovery index reachable from its subtree via tree edges plus **at most one** back/cross edge to a vertex still on the stack. Vertices are pushed onto an explicit stack as they're discovered; when a vertex `u` satisfies `low[u] === disc[u]`, it is the **root of an SCC**, and everything above `u` on the stack (down to and including `u`) forms that component.

```ts
function tarjan(adj: number[][], V: number): number[][] {
  const disc = new Array(V).fill(-1), low = new Array(V).fill(0);
  const onStack = new Array(V).fill(false), stack: number[] = [], sccs: number[][] = [];
  let idx = 0;
  const dfs = (u: number) => {
    disc[u] = low[u] = idx++; stack.push(u); onStack[u] = true;
    for (const v of adj[u]) {
      if (disc[v] === -1) { dfs(v); low[u] = Math.min(low[u], low[v]); }
      else if (onStack[v]) low[u] = Math.min(low[u], disc[v]);   // back/cross edge in-stack
    }
    if (low[u] === disc[u]) {          // u is an SCC root
      const comp: number[] = []; let w;
      do { w = stack.pop()!; onStack[w] = false; comp.push(w); } while (w !== u);
      sccs.push(comp);
    }
  };
  for (let u = 0; u < V; u++) if (disc[u] === -1) dfs(u);
  return sccs;
}
```

The `onStack` check is essential: only edges to vertices *still on the DFS stack* may lower `low`, because a vertex already popped belongs to a finished SCC and can't be part of `u`'s. One traversal, each edge relaxing `low` once — **`O(V + E)`**. Tarjan also emits SCCs in **reverse topological order** of the condensation for free.

## Kosaraju — Two Passes and a Transpose

Kosaraju is conceptually simpler at the cost of touching the graph twice:
1. Run DFS on `G`, pushing each vertex onto a stack **when it finishes** (this is the finish-time order from [[graph-traversal|DFS]]).
2. Build the **transpose** `Gᵀ` (every edge reversed).
3. Pop vertices in decreasing finish order; each DFS on `Gᵀ` from an unvisited popped vertex marks out exactly one SCC.

The insight: reversing edges preserves SCCs (mutual reachability is symmetric under reversal) but *breaks* the inter-component edges' usefulness, so a DFS in the right order can't leak out of a component. Also `O(V + E)`, but it needs `O(V + E)` extra space for the transpose and two full traversals — more cache traffic and memory than Tarjan.

## Tarjan vs Kosaraju

- **Tarjan**: single pass, no transpose, lower constant factor — the practical default, and it hands you a reverse-topological order.
- **Kosaraju**: two passes plus a transpose, but arguably easier to explain and reason about; handy when you already have `Gᵀ` or find the finish-order argument clearer.
- A third option, **Gabow's**, matches Tarjan with two stacks instead of low-link. All three are `O(V + E)`.

## Applications

- **2-SAT.** Build the implication graph; a formula is satisfiable iff no variable `x` and its negation `¬x` share an SCC — and a satisfying assignment reads off the condensation's topological order. Linear-time 2-SAT is *the* headline SCC application.
- **Deadlock / dependency-cycle detection.** A cycle among resources, modules, or microservices is a nontrivial SCC; SCC decomposition finds *all* of them at once, not just "a cycle exists."
- **Compiler and build analysis** — mutually recursive functions/modules, and collapsing them to schedule remaining work via the condensation DAG.

## Senior Pitfalls

- SCCs are **directed-only**. In an undirected graph the analogue is ordinary connected components ([[graph-traversal]] or [[union-find]]) — don't conflate them.
- In Tarjan, updating `low[u]` with `low[v]` for a **cross/forward edge** (instead of `disc[v]`, and only when `onStack`) is a classic correctness bug that merges components incorrectly.
- Both algorithms are **recursive DFS** at heart — the same deep-graph **stack-overflow** risk applies; convert to an explicit stack for millions of vertices.
- Kosaraju's transpose **doubles memory**; on huge graphs prefer Tarjan.

## See Also
- [[graph-traversal]] — DFS discovery/finish times and edge classification, the engine here
- [[topological-sort]] — orders the condensation DAG of SCCs
- [[graphs]] — directed graphs, reachability, and the condensation
- [[union-find]] — the undirected-connectivity counterpart
