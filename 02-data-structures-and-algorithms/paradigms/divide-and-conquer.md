# Divide & Conquer

Divide & conquer is an algorithm-design paradigm that solves a problem by breaking it into smaller **independent** instances of the same problem, solving those recursively, and stitching the partial answers back together. It is the structural template behind mergesort, quicksort, binary search, fast multiplication, and countless geometric algorithms — and its running time reads straight off a [[recurrence-relations|recurrence]].

## The Three-Phase Schema

Every divide-and-conquer algorithm has the same skeleton:

1. **Divide** — split the input of size `n` into `a` subproblems, typically of size `n/b`.
2. **Conquer** — solve each subproblem recursively; stop at a base case small enough to solve directly.
3. **Combine** — merge the subproblem solutions into a solution for the original.

```ts
function solve<T>(input: T): Result {
  if (isBaseCase(input)) return solveDirectly(input);   // conquer trivially
  const parts = divide(input);                          // divide
  const solved = parts.map(solve);                      // conquer recursively
  return combine(solved);                               // combine
}
```

The work outside the recursive calls — the cost of `divide` plus `combine` — is the `f(n)` term. This gives the canonical recurrence `T(n) = a·T(n/b) + f(n)`, which the **Master Theorem** solves in one step. The three master-theorem cases correspond to *where the work concentrates*: at the leaves (the recursion dominates), evenly across all `log_b n` levels, or at the root (the combine step dominates). Knowing which case you're in tells you exactly which phase to optimize.

## The Combine Step Is Where the Design Lives

The divide step is usually trivial (cut the array in half); the *combine* step is what distinguishes a clever algorithm from a naive one, and it is where most of the difficulty and the cost hide.

- **Mergesort** does a trivial split but a `Θ(n)` merge — combine is the whole algorithm. Recurrence `2T(n/2) + Θ(n) = Θ(n log n)`.
- **Quicksort** inverts this: the *partition* (a divide-time step) does the `Θ(n)` work, and combine is free (`O(1)` — the pieces are already in place). Average `Θ(n log n)`; worst `Θ(n²)` on a bad pivot.
- **Binary search** discards one half entirely, so `a = 1`: `T(n) = T(n/2) + Θ(1) = Θ(log n)`. There is no combine at all.
- **Strassen's matrix multiply** cuts an `n×n` product into 7 (not 8) subproducts of half-size matrices plus `Θ(n²)` additions: `7T(n/2)+Θ(n²) = Θ(n^{log₂7}) ≈ Θ(n^{2.807})`, beating naive `Θ(n³)`.
- **Karatsuba multiplication** multiplies two `n`-digit numbers with 3 half-size multiplies instead of 4: `3T(n/2)+Θ(n) = Θ(n^{log₂3}) ≈ Θ(n^{1.585})`, beating schoolbook `Θ(n²)`.
- **Closest pair of points** splits by a median x-line, recurses on each half, then the clever combine only checks points within `δ` of the dividing strip (at most 7 neighbors each): `2T(n/2)+Θ(n) = Θ(n log n)`, versus `Θ(n²)` brute force.

Strassen and Karatsuba are the paradigm at its most surprising: an *algebraic identity* reduces `a` (the branching factor), and because `a` sits in the exponent `log_b a`, shaving one subproblem changes the asymptotic class.

## Contrast with Dynamic Programming

The defining word above is **independent**. Divide-and-conquer subproblems don't overlap — the left half and right half of a mergesort share no work, so recomputation is impossible and there is nothing to cache. When subproblems *do* overlap (naive Fibonacci recomputes the same values exponentially often), you are no longer in divide-and-conquer territory: you want [[dynamic-programming]], which memoizes overlapping subproblems. The mental test: *draw the recursion tree — if the same subproblem node appears more than once, use DP; if every node is distinct, plain divide-and-conquer is optimal and adding a cache buys nothing but overhead.*

## Natural Parallelism

Because the subproblems are independent, they can run on separate cores with no synchronization until the combine step — divide-and-conquer is *the* structural source of parallelism (fork/join, `parallel_for`, map-reduce all descend from it). The parallel speedup is bounded by the combine work: if combine is `Θ(n)` and sequential, it becomes the critical path (Amdahl's law in miniature). Algorithms with cheap or associative combines (parallel prefix-sum, parallel reduce) parallelize best.

## Senior Pitfalls

- **Base-case size matters in practice.** Recursing down to `n = 1` is asymptotically fine but cache-hostile; real sorts cut over to insertion sort around `n ≈ 16` (see [[07-performance-engineering/data-locality|Data Locality]]). The Big-O hides this constant.
- **Recursion depth.** Balanced splits give `Θ(log n)` stack depth; a degenerate split (quicksort worst case) gives `Θ(n)` and can overflow the stack. Recurse on the smaller half first and loop on the larger to bound depth.
- **Combine cost dominates silently.** If your combine is accidentally `Θ(n log n)` instead of `Θ(n)`, you've pushed a clean `Θ(n log n)` up to `Θ(n log²n)` — always cost the combine explicitly.

## See Also

- [[recurrence-relations]]
- [[sorting-algorithms]]
- [[dynamic-programming]]
- [[recursion]]
- [[binary-search]]
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]
