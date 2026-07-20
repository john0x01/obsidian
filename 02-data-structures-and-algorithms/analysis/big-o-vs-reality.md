# Big-O vs Reality

Big-O tells you how cost *scales*; it deliberately discards everything that determines how fast your code *actually runs* on a real machine at a real input size. This note is the counterweight to [[asymptotic-notation]]: it catalogs the ways asymptotics mislead in practice, so you know when to trust the exponent and when to reach for a profiler instead.

## What Big-O Throws Away

Big-O drops constant factors and lower-order terms and assumes `n → ∞`. Every one of those is a place reality hides:

- **Constant factors are real.** Two `O(n)` algorithms can differ 10–100×. A `Θ(n log n)` algorithm with a huge constant loses to a `Θ(n²)` one until `n` is large. The crossover point, not the asymptote, decides which you should ship.
- **The uniform-cost model is a fiction.** Big-O counts operations as if every memory access costs the same. On real hardware, an access can cost ~1 cycle (L1) or ~200+ cycles (main memory) — a two-orders-of-magnitude spread the model cannot see. See [[07-performance-engineering/big-o-in-practice|Big-O in Practice]].

## The Memory Hierarchy Dominates

This is the single biggest reason "same-O" algorithms diverge. Modern CPUs are starved for data: registers → L1 → L2 → L3 → RAM span roughly 1 → 4 → 12 → 40 → 200+ cycles. Two consequences:

- **Contiguous beats pointer-chasing.** A `Θ(n)` scan of a [[dynamic-arrays|contiguous array]] and a `Θ(n)` traversal of a [[linked-lists|linked list]] have identical Big-O, yet the array is often *5–10× faster*. The array streams through cache lines (a 64-byte line prefetched holds many elements) with a predictable access pattern the hardware prefetcher loves. The linked list dereferences a pointer to a heap node that may live anywhere, cache-missing on nearly every step — no spatial locality, no prefetch. See [[07-performance-engineering/data-locality|Data Locality]] and [[07-performance-engineering/cache-friendly-code|Cache-Friendly Code]].
- **This inverts textbook advice.** Inserting into the middle of an array is `O(n)` (shift elements) vs a linked list's `O(1)` — but if you still have to *find* the position by traversal, the array's cache-friendly shift usually wins in wall-clock time for the sizes real programs see. `std::vector` beats `std::list` for the overwhelming majority of workloads for exactly this reason.

## Branch Prediction

The CPU speculatively executes past branches; a misprediction flushes the pipeline (~15–20 cycles). Algorithms with *unpredictable* branches pay a cost invisible to Big-O. The famous example: summing an array runs dramatically faster when it is *sorted*, because a `if (x > threshold)` branch becomes perfectly predictable. Branch-free (`branchless`) code — using arithmetic or conditional-move instead of a branch — can beat "fewer operations" code that branches unpredictably.

## Small-n Regimes and Timsort's Cutoff

For small `n`, the algorithm with the better asymptote frequently *loses*:

- **Insertion sort is `Θ(n²)`** but has tiny constants, no recursion overhead, sequential access, and is adaptive (near-`O(n)` on nearly-sorted data). **Mergesort is `Θ(n log n)`** but pays allocation, recursion, and merge overhead.
- Below a threshold (commonly `n ≈ 16–64`), insertion sort wins outright. That is why production sorts are **hybrids**: [[sorting-algorithms|Timsort]] (Python, Java objects) and introsort (C++ `std::sort`) recurse with a fast divide-and-conquer down to small runs, then switch to insertion sort for the leaves. The cutoff is a measured constant, not a theoretical one — pure asymptotics would never suggest it. See [[recurrence-relations]] for why the divide-and-conquer part is `Θ(n log n)`.

```ts
// Real hybrid sorts do exactly this at the leaves.
function sort(a: number[], lo: number, hi: number) {
  if (hi - lo < 32) return insertionSort(a, lo, hi); // small n: better constants win
  const mid = lo + ((hi - lo) >> 1);
  sort(a, lo, mid); sort(a, mid + 1, hi);
  merge(a, lo, mid, hi);
}
```

## Galactic Algorithms

At the far extreme, some algorithms have the *best known asymptotic* bound but are useless in practice because their constants are astronomically large — they only win for inputs larger than anything physically representable. The Coppersmith–Winograd family for matrix multiplication (`O(n^~2.37)`) is "galactic": nobody runs it; practical code uses Strassen (`Θ(n^2.807)`) or cache-blocked naive multiply. Big-O crowned a winner that reality will never see.

## Senior Takeaways

- Use Big-O to reject algorithmically-doomed designs (`O(n²)` on `n = 10⁷`) — it is authoritative for *scaling*.
- **Measure** to choose among same-class options; the constant, the cache pattern, and the input distribution decide the winner. See [[07-performance-engineering/algorithmic-optimization|Algorithmic Optimization]].
- Prefer contiguous, predictable, cache-friendly layouts; a worse asymptote with better locality often wins at real sizes.
- Know your `n`. The crossover point is the real decision boundary — profile at *your* input sizes, not `n → ∞`.

## See Also

- [[asymptotic-notation]]
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]
- [[07-performance-engineering/data-locality|Data Locality]]
- [[07-performance-engineering/cache-friendly-code|Cache-Friendly Code]]
- [[07-performance-engineering/algorithmic-optimization|Algorithmic Optimization]]
- [[sorting-algorithms]]
