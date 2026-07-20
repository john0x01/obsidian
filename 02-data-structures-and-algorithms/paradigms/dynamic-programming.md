# Dynamic Programming

Dynamic programming (DP) solves a problem by breaking it into overlapping subproblems, solving each **once**, and reusing the stored answer. It converts an exponential recursion that recomputes the same subproblems into a polynomial one. DP applies exactly when a problem has *optimal substructure* and *overlapping subproblems* — the two conditions are the entire test for whether DP is the right tool.

## The Two Preconditions

1. **Optimal substructure** — an optimal solution is composed of optimal solutions to subproblems. (Shared with [[divide-and-conquer]] and [[greedy-algorithms]].)
2. **Overlapping subproblems** — the recursion revisits the same subproblem many times. This is what distinguishes DP from divide-and-conquer: mergesort's halves never overlap, so caching them is pointless; naive Fibonacci recomputes `fib(k)` exponentially often, so caching collapses `Θ(φⁿ)` to `Θ(n)`.

If subproblems don't overlap, use divide-and-conquer. If a locally optimal choice is provably safe, use greedy (cheaper). DP is the middle ground: overlapping subproblems where you *must* consider multiple choices per state.

## Memoization vs Tabulation

The same recurrence can be evaluated two ways:

```ts
// Top-down: recursion + cache. Computes only reachable states, lazily.
const memo = new Map<number, number>();
function f(n: number): number {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n)!;
  const r = f(n - 1) + f(n - 2);
  memo.set(n, r);
  return r;
}

// Bottom-up: fill a table in dependency order. No recursion, no cache misses.
function fib(n: number): number {
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) [a, b] = [b, a + b];
  return n === 0 ? 0 : b;
}
```

**Top-down (memoization)** mirrors the recursive definition — write the recurrence, add a cache, done. It computes *only the states you actually reach*, which is a real win when the reachable set is sparse. Costs: recursion-stack depth (risk of overflow on deep chains) and per-call hashing/function-call overhead. See [[07-performance-engineering/memoization|Memoization]] and [[recursion]].

**Bottom-up (tabulation)** fills a table in an explicit dependency order. No recursion overhead, cache-friendly sequential access (see [[07-performance-engineering/data-locality|Data Locality]]), and it enables **space optimization** because you control iteration order. Cost: you must find a valid topological order of states by hand, and you may compute states you never needed. For most interview/production DP, tabulation is faster by a constant factor; memoization is faster to write and safer when the state space is sparse or irregular.

## State Design and the Complexity Formula

DP is *state design*. A state is the minimal set of parameters that fully determines a subproblem's answer; the **transition** is the recurrence relating a state to previously-computed states. Get the state wrong (too few parameters) and the recurrence is incorrect; too many and it blows up.

> **Total time = (number of states) × (cost per transition).**

This single formula lets you predict complexity before writing code. LCS has `Θ(mn)` states each with `O(1)` transition → `Θ(mn)`. A DP with `O(n)` states but an `O(n)` inner loop per transition is `Θ(n²)`. Interval DP with `O(n²)` states and an `O(n)` split point is `Θ(n³)`. Knowing this tells you immediately whether a formulation is fast enough — and where to attack it (fewer states, or cheaper transition via a [[monotonic-stack-and-queue|monotonic structure]], prefix sums, or a segment tree).

## Classic Patterns

- **0/1 knapsack** — state `(item index, remaining capacity)`, `Θ(nW)` (pseudo-polynomial). The canonical case where [[greedy-algorithms|greedy]] fails.
- **LCS / edit distance** — 2-D grid over the two string prefixes, `Θ(mn)`.
- **Longest increasing subsequence (LIS)** — `Θ(n²)` DP, or `Θ(n log n)` with a patience-sorting [[binary-search]].
- **Coin change** — min coins / count ways, `Θ(amount × coins)`.
- **Interval DP** — matrix-chain multiplication, optimal BST; states are `(i, j)` ranges, transition splits at `k`, `Θ(n³)`.
- **Tree DP** — DP over subtrees (e.g. max independent set on a tree), one state per node.
- **Bitmask DP** — subset as an integer, `Θ(2ⁿ · n)`; Traveling Salesman for small `n`. See [[bit-manipulation]].
- **[[kadanes-algorithm|Kadane's algorithm]]** — the minimalist DP: one scalar state (best subarray ending here), `Θ(n)`.

## Space Optimization via Rolling Arrays

When a transition only reads the previous row (or few rows), you don't need the full table. 0/1 knapsack drops from `Θ(nW)` to `Θ(W)` by iterating capacity **downward** in place (upward would reuse the *current* item, silently turning it into unbounded knapsack — a classic bug). Fibonacci needs two scalars, not an array. The trade-off: rolling arrays destroy the information needed for **path reconstruction** — if you must recover *which* items/characters formed the optimum, keep the full table or store parent pointers.

## Senior Pitfalls

- **Missing a state dimension** — the recurrence looks right but double-counts or references undefined subproblems. Verify the state captures *everything* the transition depends on.
- **Wrong iteration direction** in space-optimized 0/1 knapsack (see above).
- **Pseudo-polynomial ≠ polynomial** — knapsack's `Θ(nW)` is exponential in the *bit length* of `W`; that's why 0/1 knapsack is NP-hard despite the tidy table.
- **Memoization stack overflow** on long dependency chains — convert to bottom-up.

## See Also

- [[07-performance-engineering/memoization|Memoization]]
- [[divide-and-conquer]]
- [[greedy-algorithms]]
- [[recursion]]
- [[kadanes-algorithm]]
- [[recurrence-relations]]
