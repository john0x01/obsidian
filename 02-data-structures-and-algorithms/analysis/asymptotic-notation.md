# Asymptotic Notation

Asymptotic notation is the vocabulary for describing how an algorithm's cost grows as its input grows without bound. It solves a real problem: comparing algorithms in a way that is independent of the machine, the language, and the constant factors that vary between them — so you can reason about *scalability* rather than a single benchmark.

## The Three Bounds (and What They Actually Mean)

The bounds are statements about functions, not about programs. Let `T(n)` be the cost function you care about (comparisons, operations, whatever). Each notation is a *set* of functions:

- **Big-O — asymptotic upper bound.** `T(n) = O(g(n))` iff there exist constants `c > 0` and `n₀` such that `T(n) ≤ c·g(n)` for all `n ≥ n₀`. It caps growth from above: "no worse than."
- **Big-Ω — asymptotic lower bound.** `T(n) = Ω(g(n))` iff `T(n) ≥ c·g(n)` for all `n ≥ n₀`. It floors growth: "no better than."
- **Big-Θ — tight bound.** `T(n) = Θ(g(n))` iff it is *both* `O(g(n))` and `Ω(g(n))`. This pins the growth rate exactly (up to constants).

The `n₀` matters: these are claims about *eventually*, which is why small-`n` behavior is legitimately ignored (see [[big-o-vs-reality]] for why that abstraction can bite).

## Best / Average / Worst Is a Separate Axis

A pervasive confusion: people conflate "worst case" with "Big-O" and "best case" with "Big-Ω." They are orthogonal. Best/average/worst-case describes *which input* you measure; O/Ω/Θ describes *which side you bound* the resulting function.

You can legitimately say all of these about one algorithm. Quicksort's worst case is `Θ(n²)` and its average case is `Θ(n log n)` — both are *tight* bounds, each on a different input distribution. It is perfectly valid (if weak) to say quicksort's worst case is `O(n³)`: an upper bound need not be tight. So "worst-case `O(...)`" answers *two* questions: worst input, and an upper bound on the cost for that input.

## The Growth-Class Ladder

Ordered from slowest- to fastest-growing:

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!)
```

To compare two growth classes, take the limit of their ratio as `n → ∞`. If `f(n)/g(n) → 0`, then `f = O(g)` but not `Θ(g)` (f is strictly smaller). If it → a positive constant, they are `Θ` of each other. Useful facts: any polynomial beats any polylogarithm (`n^ε` dominates `(log n)^k` for any `ε > 0`), and any exponential beats any polynomial. Logarithm bases vanish into the constant — `log₂ n` and `log₁₀ n` differ by a constant factor, so we never write the base inside O.

## Deriving Bounds From Code

- **A single loop over `n`** → `O(n)`.
- **Nested loops** multiply: two loops each to `n` → `O(n²)`; but a loop where the inner bound depends on the outer (`for j = i..n`) sums to `n + (n-1) + ... = Θ(n²)` — still quadratic, half the constant.
- **Halving** each iteration → `O(log n)`. The loop below runs `⌊log₂ n⌋ + 1` times because `n` is divided until it hits the base case.

```ts
function countHalvings(n: number): number {
  let steps = 0;
  while (n > 1) { n = Math.floor(n / 2); steps++; } // log n iterations
  return steps;
}
```

- **Sequential blocks add**, then you keep the dominant term: `O(n) + O(n²) = O(n²)`.
- **Divide-and-conquer** yields a recurrence — solve it with the Master Theorem; see [[recurrence-relations]].

## Senior Pitfalls & Misconceptions

- **"O is the runtime."** No — O is an *upper bound* on a growth rate, not a precise cost and not necessarily tight. Reaching for Θ when you mean "exactly this rate" is more honest.
- **Forgetting to drop constants and lower-order terms.** `3n² + 100n + 7` is `Θ(n²)`. Writing `O(3n²)` is not wrong but is non-idiomatic; the constant is meaningless asymptotically.
- **Worst vs average confusion.** Hash-table lookup is `O(1)` average but `O(n)` worst case under adversarial collisions; quoting only the average hides a real failure mode. See [[hash-tables]].
- **`O(1)` hiding a huge constant.** "Constant time" can mean a 10,000-cycle operation. Two `O(n)` algorithms can differ 50× from cache behavior and branch prediction alone — asymptotics say nothing about the machine.
- **Ignoring the variable's meaning.** For graphs, `O(V + E)` and `O(V²)` diverge wildly depending on density; be explicit about *what* `n` counts.
- **Amortized ≠ asymptotic-per-op.** A single dynamic-array push is `O(n)` worst case but `O(1)` amortized; see [[amortized-analysis]].

## See Also

- [[amortized-analysis]]
- [[recurrence-relations]]
- [[space-complexity]]
- [[big-o-vs-reality]]
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]
- [[sorting-algorithms]]
