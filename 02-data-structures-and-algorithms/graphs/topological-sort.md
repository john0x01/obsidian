# Topological Sort

A **topological ordering** of a directed acyclic graph is a linear arrangement of its vertices such that for every edge `u→v`, `u` appears before `v`. It answers "in what order can I do these tasks so that every prerequisite comes first?" — the abstract shape of build systems, schedulers, and dependency resolvers. A topological order **exists if and only if the graph is a [[graphs|DAG]]**; a directed cycle makes the ordering impossible, because the cycle's vertices would each need to precede the other.

## The Core Fact

Any DAG has at least one topological order (often many — the ordering is not unique). Both standard algorithms are `O(V + E)` — a topological sort is essentially a traversal that emits vertices at the right moment.

## Kahn's Algorithm — BFS on In-Degrees

The intuition: a vertex with **in-degree 0** has no unmet prerequisites, so it can go next. Emit it, remove it (decrementing its neighbours' in-degrees), and repeat.

```ts
function kahn(adj: number[][], V: number): number[] {
  const indeg = new Array(V).fill(0);
  for (const nbrs of adj) for (const v of nbrs) indeg[v]++;
  const q = adj.map((_, u) => u).filter(u => indeg[u] === 0);
  const order: number[] = [];
  for (let i = 0; i < q.length; i++) {
    const u = q[i]; order.push(u);
    for (const v of adj[u]) if (--indeg[v] === 0) q.push(v);
  }
  return order;              // order.length < V  ⟺  a cycle exists
}
```

Computing in-degrees is `O(V + E)`; the main loop processes each vertex once and each edge once, also `O(V + E)`. **Cycle detection falls out for free**: if the emitted order contains fewer than `V` vertices, the leftover vertices all sit on or downstream of a cycle (they never reached in-degree 0). Kahn's also naturally supports **lexicographically smallest** or priority-ordered output — swap the [[queues|queue]] for a [[priority-queues|min-heap]] and you get the smallest valid schedule, useful when ties must break deterministically.

## DFS Finish-Time Reversal

The alternative uses [[graph-traversal|DFS]] and the parenthesis structure of finish times. Run DFS; when a vertex **finishes** (all its descendants are done), push it onto a stack. The **reverse of finish order** is a valid topological order.

```ts
function dfsTopo(adj: number[][], V: number): number[] {
  const state = new Array(V).fill(0);   // 0=unseen 1=on-stack 2=done
  const out: number[] = [];
  const go = (u: number) => {
    state[u] = 1;
    for (const v of adj[u]) {
      if (state[v] === 1) throw new Error("cycle");  // back edge
      if (state[v] === 0) go(v);
    }
    state[u] = 2; out.push(u);
  };
  for (let u = 0; u < V; u++) if (state[u] === 0) go(u);
  return out.reverse();
}
```

Why it works: when `u` finishes, every vertex reachable from `u` has already finished and been pushed, so `u` lands *before* them all after reversal — exactly the topological invariant. The **three-colour state** (unseen / on the recursion stack / fully done) is what makes cycle detection precise: encountering a vertex that is still *on the stack* is a **back edge**, the definitive signature of a directed cycle. Merely "already visited" is not enough — that could be a harmless cross or forward edge into a finished subtree.

## Kahn vs DFS

- **Kahn's** is iterative (no recursion-depth risk), reports cycles by a simple count, and slots naturally into priority-ordered scheduling. Preferred when you want the cycle *members* or a deterministic tie-break.
- **DFS** is a few lines shorter, shares machinery with SCC and other DFS algorithms, but risks stack overflow on deep graphs and needs the three-colour trick to detect cycles correctly.

Both are correct and `O(V + E)`; the choice is stylistic and operational, not asymptotic.

## Applications

- **Build systems / task runners** (Make, Bazel, npm scripts): compile targets after their dependencies.
- **Dependency resolution**: package managers ordering installs, spreadsheet cell recalculation, module load order.
- **Scheduling with precedence constraints** and course-prerequisite planning.
- A cheap **cycle check for directed graphs** as a byproduct — "does my dependency graph have a cycle?" is the same question as "does a topological order exist?"

## Senior Pitfalls

- **A topological order is not unique.** Never assume a canonical answer unless you deliberately impose a tie-break (lexicographic via a heap). Tests that hard-code one ordering are brittle.
- **Applying it to a non-DAG.** You must handle the cycle case; silently returning a partial order (Kahn's) or looping (naïve DFS without the on-stack check) is a classic bug.
- **Undirected graphs have no topological order** — the concept is meaningless without edge direction.
- Recursive DFS shares the deep-graph **stack-overflow** hazard of all DFS; prefer Kahn's or an explicit stack for very large or adversarial inputs.

## See Also
- [[graph-traversal]] — DFS finish times and edge classification, the mechanism here
- [[graphs]] — DAGs, in-degree, and directed-cycle definitions
- [[priority-queues]] — for lexicographic / priority-ordered topological output
- [[strongly-connected-components]] — condense cycles first, then topologically sort the DAG of SCCs
- [[dynamic-programming]] — DP over a DAG processes states in topological order
