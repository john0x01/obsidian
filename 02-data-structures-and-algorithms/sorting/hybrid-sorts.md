# Hybrid Sorts (Timsort & Introsort)

No production standard library sorts with a textbook algorithm. They use **hybrids** that combine two or three sorts to get the best-case speed of one, the worst-case guarantee of another, and low constants for small inputs. The two that matter: **Timsort** (Python's `sorted`, Java's `Arrays.sort` for objects, V8/JavaScript's `Array.prototype.sort`) and **Introsort** (C++ `std::sort`). The unifying idea: **detect conditions where your primary sort is weak, and fall back.**

## Why not a textbook sort?

Every pure sort has a fatal flaw for a general-purpose library:

- [[quick-sort]]: fastest on average but `Θ(n²)` on adversarial/sorted input, and unstable.
- [[merge-sort]]: stable and worst-case `Θ(n log n)` but needs `Θ(n)` scratch and doesn't exploit pre-sorted data.
- [[heap-sort]]: `O(1)` space, no bad case, but poor cache locality → slow in practice.
- [[insertion-sort]]: `Θ(n²)` in general but **unbeatable for small or nearly-sorted `n`** with tiny constants and stability.

A hybrid stitches these together so the composite has no single fatal case.

## Timsort — merge sort that exploits real-world order

Timsort (Tim Peters, 2002) is an **adaptive, stable** merge sort built on the observation that real data has structure — pre-existing sorted or reverse-sorted stretches.

**Mechanism:**
1. **Find natural runs.** Scan left to right for maximal ascending (or strictly descending, then reverse in place) runs. Real data often already contains long runs.
2. **Extend short runs with insertion sort.** Runs shorter than a `minrun` threshold (32–64) are extended and sorted with **binary insertion sort** — cheap for small `n`.
3. **Merge runs** with [[merge-sort]]'s stable merge, but push runs on a stack and merge under **invariants** (e.g. `len[i] > len[i+1] + len[i+2]`) that keep merges balanced.
4. **Galloping mode.** When one run keeps "winning" the merge comparison, Timsort switches to exponential/binary search to skip a chunk of that run at once, turning a run of `m` consecutive wins from `m` comparisons into `O(log m)`. This is what makes merging two runs with a long clustered overlap fast.

**Properties:** **stable**; worst case `Θ(n log n)`; space `O(n)`; and the killer feature — **`Θ(n)` on already-sorted (or reverse-sorted) input**, because it's discovered as one giant run and no merging is needed. That adaptivity is why it's the default for object sorting in Python, Java, and JS.

```ts
// Sketch (not the real thing): the two decisions Timsort makes.
const MINRUN = 32;
// for each run:
//   run = longest ascending/descending prefix (reverse if descending)
//   if (run.length < MINRUN) extend to MINRUN and binaryInsertionSort(run)
//   pushToStack(run); mergeCollapse();   // merge while stack invariants violated
```

## Introsort — quicksort with a safety net

Introsort (David Musser, 1997), the guts of C++ `std::sort`, keeps quicksort's speed but **eliminates its `Θ(n²)` worst case** with a depth guard.

**Mechanism:**
1. Run **quicksort** (usually median-of-three pivot) — fast average case, cache-friendly.
2. Track recursion **depth**. If it exceeds `2·⌊log₂ n⌋` — the signature of degenerate, unbalanced partitions heading toward `Θ(n²)` — **bail out to [[heap-sort]]** on that subarray. Heap sort finishes it in guaranteed `Θ(n log n)` with `O(1)` space.
3. When a subarray falls below ~16 elements, stop recursing and finish with a single **insertion sort** pass over the whole array (nearly-sorted by then, so insertion sort is near-linear).

**Properties:** worst case **`Θ(n log n)` guaranteed** (heap-sort fallback), average as fast as quicksort, `O(log n)` space, **not stable** (quicksort and heapsort aren't). C++ offers `std::stable_sort` (a merge-sort hybrid) when stability is needed.

## Dry-run intuition

- **Timsort on `[1,2,3,4,10,9,8,7]`:** finds ascending run `[1,2,3,4]`, then descending `[10,9,8,7]` which it reverses to `[7,8,9,10]`, then one stable merge → `Θ(n)` total, essentially no work beyond two scans.
- **Introsort on sorted `[1..n]` with median-of-three:** median-of-three picks a good pivot so partitions stay balanced; even if they didn't, the depth counter would trip and heap-sort would cap the cost at `Θ(n log n)` — never the `Θ(n²)` a naive quicksort would suffer here.

## Complexity summary

| | best | average | worst | space | stable |
|---|---|---|---|---|---|
| **Timsort** | `Θ(n)` | `Θ(n log n)` | `Θ(n log n)` | `O(n)` | yes |
| **Introsort** | `Θ(n log n)` | `Θ(n log n)` | `Θ(n log n)` | `O(log n)` | no |

## The general "detect-and-fall-back" pattern

Both encode the same design principle: **combine complementary algorithms and switch based on a runtime signal.**
- Small `n` → insertion sort (low constants, cache-local) — both use this.
- Pre-sorted structure → exploit runs / good pivots (Timsort's runs, Introsort's median-of-three).
- Worst-case danger detected → fall back to a guaranteed algorithm (Introsort's depth-limit → heap sort). Timsort has no `Θ(n²)` mode to guard against because merge sort is already worst-case safe.

This is the same "measure, then choose the right tool" thinking behind [[07-performance-engineering/algorithmic-optimization|algorithmic optimization]] generally.

## When to use / when not

You rarely implement these — you *use* them via the standard library, and knowing which one your language ships tells you the guarantees: **stable + adaptive** (Timsort: Python/Java-objects/JS) or **fast + worst-case-safe but unstable** (Introsort: C++). Reach for a hand-rolled sort only for special key structure ([[radix-sort]]/[[counting-sort]]) or memory constraints.

## Interview pitfalls & gotchas

- **Thinking the stdlib uses plain quicksort/merge sort** — naming Timsort/Introsort and *why* is the differentiator.
- **Assuming `Array.prototype.sort` is unstable** — it was engine-dependent pre-2019, but ES2019 **mandates stability**; V8 uses Timsort.
- **Forgetting Introsort is unstable** — reach for `stable_sort` in C++ when tie-order matters.
- **Missing that Timsort is `Θ(n)` on sorted data** — the adaptivity is the whole point; a claim of "always `n log n`" understates it.
- **Ignoring the insertion-sort cutoff** — the small-`n` fallback is a real, measurable win, not a detail.

## See Also
- [[merge-sort]]
- [[quick-sort]]
- [[heap-sort]]
- [[insertion-sort]]
- [[sorting-algorithms]]
