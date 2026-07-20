# Sorting Algorithms

Sorting is the most-studied problem in computing because so many others reduce to it: binary search, deduplication, order statistics, and near-duplicate detection all become easy once data is ordered. This note is the **map of the area** — the comparison table, the `Ω(n log n)` lower bound that caps every comparison sort, precise definitions of stability and in-place, and a decision procedure for picking one. Each algorithm has its own note; this one links out.

## The Comparison Table

| Algorithm | Best | Average | Worst | Space (aux) | Stable | In-place |
|---|---|---|---|---|---|---|
| [[bubble-sort]] | `Θ(n)`¹ | `Θ(n²)` | `Θ(n²)` | `O(1)` | ✅ | ✅ |
| [[selection-sort]] | `Θ(n²)` | `Θ(n²)` | `Θ(n²)` | `O(1)` | ❌² | ✅ |
| [[insertion-sort]] | `Θ(n)` | `Θ(n²)` | `Θ(n²)` | `O(1)` | ✅ | ✅ |
| [[shell-sort]] | `Θ(n log n)` | gap-dependent | `Θ(n²)`–`Θ(n^1.3)` | `O(1)` | ❌ | ✅ |
| [[merge-sort]] | `Θ(n log n)` | `Θ(n log n)` | `Θ(n log n)` | `O(n)` | ✅ | ❌ |
| [[quick-sort]] | `Θ(n log n)` | `Θ(n log n)` | `Θ(n²)`³ | `O(log n)` | ❌ | ✅ |
| [[heap-sort]] | `Θ(n log n)` | `Θ(n log n)` | `Θ(n log n)` | `O(1)` | ❌ | ✅ |
| [[counting-sort]] | `Θ(n+k)` | `Θ(n+k)` | `Θ(n+k)` | `O(n+k)` | ✅ | ❌ |
| [[radix-sort]] | `Θ(d(n+b))` | `Θ(d(n+b))` | `Θ(d(n+b))` | `O(n+b)` | ✅ | ❌ |
| [[bucket-sort]] | `Θ(n+k)` | `Θ(n+k)`⁴ | `Θ(n²)` | `O(n+k)` | ✅⁴ | ❌ |

¹ with early-exit flag. ² typical min-select impl (see below). ³ randomized/introsort avoids it. ⁴ uniform input, stable inner sort.

The top eight are **comparison sorts** (they only ever ask "is `a < b`?"). The bottom three are **non-comparison sorts** that read the keys' structure to index directly — that is how they dodge the lower bound below.

## The Ω(n log n) Comparison Lower Bound

**Any** algorithm that learns about the data *only through pairwise comparisons* needs `Ω(n log n)` comparisons in the worst case. The proof is the **decision-tree argument**:

Model an execution as a binary tree. Each internal node is one comparison `a < b`; the two children are the "yes"/"no" branches; each leaf is the permutation the algorithm outputs on reaching it.

```
                a<b ?
              /       \
          b<c?         a<c?     ← each node = one comparison
         /    \       /    \
     [abc]  a<c?   [bac]  b<c?  ← each leaf = one output permutation
```

For the sort to be correct, **every** one of the `n!` input permutations must be untangled into sorted order, and distinct permutations require distinct leaves — so the tree has `≥ n!` leaves. A binary tree of height `h` has at most `2^h` leaves, so `2^h ≥ n!`, giving `h ≥ log₂(n!)`. The height **is** the longest root-to-leaf path — the worst-case comparison count. By Stirling, `log₂(n!) ≈ n log₂ n − n log₂ e = Θ(n log n)`.

So merge sort and heap sort are **asymptotically optimal** — no comparison sort can beat them in order of growth. See [[recurrence-relations]] for solving the `2T(n/2)+Θ(n)` recurrence that reaches the same `Θ(n log n)` from above.

## Two Definitions Everyone Confuses

- **Stable** — equal keys keep their original relative order. It matters when you sort by a secondary key first, then a primary key: stability preserves the secondary order *within* equal primaries (sort by name, then by department → alphabetical within each department, for free). Merge, insertion, bubble, counting, radix are stable; selection, shell, quick, heap are not without extra work.
- **In-place** — uses `O(1)` auxiliary memory beyond the input (the recursion stack, e.g. quicksort's `O(log n)`, is conventionally tolerated). Quick, heap, and the `Θ(n²)` sorts are in-place; classic merge sort is not — it needs an `O(n)` scratch buffer to merge.

Stability and in-place-ness usually **trade off**: the stable sorts either use extra space (merge) or are slow (insertion). That tension is exactly why standard libraries pick different algorithms for primitives versus objects.

## How to Choose a Sort

Walk this checklist top to bottom; the first match wins:

1. **Is `n` tiny (≲ 16–32)?** Use [[insertion-sort]]. Its constant factors and cache behaviour beat everything asymptotically faster at small sizes — which is why the hybrids below fall back to it.
2. **Nearly sorted / mostly-in-order stream?** [[insertion-sort]] (`O(n)` here) or an adaptive merge (Timsort). Insertion sort is also **online** — it can sort a stream as elements arrive.
3. **Integer / fixed-width keys in a bounded range?** Skip comparisons entirely: [[counting-sort]] (`k = O(n)`) or [[radix-sort]] (wide keys). [[bucket-sort]] for uniformly-distributed reals.
4. **Need guaranteed `O(n log n)` and stability?** [[merge-sort]] — the cost is `O(n)` space. Ideal for linked lists and external/on-disk data (merging is sequential).
5. **Need guaranteed `O(n log n)` and `O(1)` space (stability not required)?** [[heap-sort]] — but expect poor cache locality (see [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]).
6. **General-purpose, fastest average, in-place, stability irrelevant?** [[quick-sort]] with a randomized/median-of-three pivot.

Deciding factors, in order: **key type** (comparison vs integer), **`n`**, **memory budget**, **stability requirement**, **worst-case guarantee**. [[bubble-sort]] and [[selection-sort]] never win this contest — they are pedagogical (though selection's `n−1` writes matter when writes are expensive).

## What Real Standard Libraries Ship

Nobody ships a textbook sort — they ship **hybrids** (full detail in [[hybrid-sorts]]):

- **Python (`sorted`/`list.sort`)**, **Java objects (`Collections.sort`)**, **JS V8 (`Array.prototype.sort`)** → **Timsort**: adaptive, *stable* merge sort that detects existing runs, `O(n)` on nearly-sorted, `O(n log n)` worst. ES2019 mandates stability, so JS relies on it.
- **C++ `std::sort`** → **introsort**: quicksort that watches recursion depth and switches to heap sort past `~2 log n`, guaranteeing `O(n log n)` while keeping quicksort's speed; cuts to insertion sort on small runs.
- **Java primitives (`Arrays.sort(int[])`)** → **dual-pivot quicksort** (stability is meaningless for primitives, in-place is faster).

The pattern: **stable Timsort for objects, unstable in-place quick/introsort for primitives.** Knowing which your language uses tells you whether you may rely on stability.

## Interview Pitfalls

- **"Quicksort is `O(n log n)`."** Only on average — sorted input with a naive pivot is `Θ(n²)`. Randomization or introsort is the fix.
- **"My sort is stable."** Depends on language *and* element type — verify before relying on secondary-key order.
- **`mid = (lo + hi) / 2` overflow** in partition/merge index math — use `lo + ((hi - lo) >> 1)`.
- **"Radix is linear, so it's fastest."** The hidden `d` factor and cache misses often make `Θ(n log n)` quicksort win for realistic `n` — a classic [[big-o-vs-reality|Big-O vs Reality]] trap.

## See Also

- [[bubble-sort]] · [[selection-sort]] · [[insertion-sort]] · [[shell-sort]]
- [[merge-sort]] · [[quick-sort]] · [[heap-sort]]
- [[counting-sort]] · [[radix-sort]] · [[bucket-sort]] · [[hybrid-sorts]]
- [[recurrence-relations]] · [[big-o-vs-reality|Big-O vs Reality]]
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]
