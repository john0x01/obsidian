# Quick Sort

Quick Sort partitions the array around a chosen **pivot** — smaller elements to the left, larger to the right — then recurses on each side. It is the default in-memory sort in many libraries because its average `Θ(n log n)` has **tiny constants** and excellent cache behavior. Its dark secret: an unlucky pivot degrades it to `Θ(n²)`.

## The idea / invariant

Partitioning does the real work; recursion is trivial. After partitioning, the pivot sits in its **final sorted position**, everything left of it is `≤` it, everything right is `≥` it. Recurse on the two sides; concatenation is implicit because sorting happens in place.

```
pivot = 4
[3 7 8 5 2 1 9 4]  ->  [3 2 1] 4 [7 8 5 9]
                        recurse    recurse
```

## Implementation (Lomuto partition, in-place)

```ts
function quickSort(a: number[], lo = 0, hi = a.length - 1): void {
  while (lo < hi) {
    const p = partition(a, lo, hi);           // p is now in final place
    // Recurse into the SMALLER side, loop on the larger:
    // bounds stack depth at O(log n) even on bad pivots.
    if (p - lo < hi - p) {
      quickSort(a, lo, p - 1);
      lo = p + 1;
    } else {
      quickSort(a, p + 1, hi);
      hi = p - 1;
    }
  }
}

function partition(a: number[], lo: number, hi: number): number {
  const pivot = a[hi];        // Lomuto: pivot is the last element
  let i = lo;                 // a[lo..i-1] hold elements <= pivot
  for (let j = lo; j < hi; j++) {
    if (a[j] <= pivot) { [a[i], a[j]] = [a[j], a[i]]; i++; }
  }
  [a[i], a[hi]] = [a[hi], a[i]];  // drop pivot into its slot
  return i;
}
```

**Lomuto** (above) keeps one boundary index `i` and swaps every `≤`-pivot element forward. **Hoare's** scheme uses two pointers converging from both ends; it does ~3× fewer swaps and handles duplicates better, but its `partition` returns a split point (not the pivot's final index), so the recursion bounds differ — a common source of off-by-one bugs.

## Dry run — one Lomuto partition

Partition `[3, 7, 8, 5, 2, 1, 4]`, pivot `= a[hi] = 4`, `i = lo = 0`:

| j | a[j] | ≤4? | action | array | i |
|---|------|-----|--------|-------|---|
| 0 | 3 | yes | swap self, i→1 | `[3,7,8,5,2,1,4]` | 1 |
| 1 | 7 | no | — | `[3,7,8,5,2,1,4]` | 1 |
| 2 | 8 | no | — | `[3,7,8,5,2,1,4]` | 1 |
| 3 | 5 | no | — | `[3,7,8,5,2,1,4]` | 1 |
| 4 | 2 | yes | swap a[1]↔a[4], i→2 | `[3,2,8,5,7,1,4]` | 2 |
| 5 | 1 | yes | swap a[2]↔a[5], i→3 | `[3,2,1,5,7,8,4]` | 3 |

Finally swap pivot into `a[i]=a[3]`: `[3,2,1,4,7,8,5]`, return `3`. The `4` is now permanently placed; `[3,2,1]` and `[7,8,5]` recurse.

## Complexity

**Time.**
- **Best / average — `Θ(n log n)`.** A balanced-ish split gives `T(n) = 2T(n/2) + Θ(n)`; averaged over random pivots the expected number of comparisons is `≈ 1.39 n log₂ n`. The `Θ(n)` per level is the partition scan.
- **Worst — `Θ(n²)`.** If every partition peels off just one element (`T(n) = T(n-1) + Θ(n)`), the sum `n + (n-1) + … = Θ(n²)`. The classic trigger: **already-sorted (or reverse-sorted) input with a fixed last/first-element pivot** — every pivot is the extreme, so partitions are maximally unbalanced. Adversarial inputs can force this deliberately.

**Space — `Θ(log n)` on average.** Quick sort is **in-place** (partitions swap within the array), so the only extra space is the recursion stack. Recursing into the **smaller side first** and iterating on the larger (as coded) caps stack depth at `Θ(log n)` even in the worst case — without that trick a naive version can blow the stack with `Θ(n)` depth on sorted input.

**Stable — NO.** Partition swaps non-adjacent elements, reordering equal keys. Making it stable requires extra arrays, forfeiting the in-place win.

## Pivot strategies (the fix for `Θ(n²)`)

- **Randomized pivot**: pick a random index and swap it to the end. Makes worst case a matter of astronomically bad luck, not input shape — expected `Θ(n log n)` on *any* input.
- **Median-of-three**: pivot = median of `a[lo]`, `a[mid]`, `a[hi]`. Cheap, kills the sorted-input worst case, improves splits on real data.
- **3-way (Dutch National Flag) partition**: split into `< pivot`, `== pivot`, `> pivot`. Equal keys are grouped and skipped by recursion, so arrays with **many duplicates** go from `Θ(n²)` to `Θ(n·k)` where `k` is distinct-value count — near-linear when duplicates dominate.

## When to use / when not

- **Use** as a general-purpose in-memory array sort: fastest average performer, `O(log n)` space, cache-friendly sequential scans.
- **Don't use** when you need a hard worst-case guarantee (real-time systems) — use [[merge-sort]] or [[heap-sort]] — or when **stability** matters. Production libraries hedge with [[hybrid-sorts|Introsort]]: quicksort that falls back to heapsort past a depth limit.
- Its partition is the engine behind [[quickselect-and-top-k|Quickselect]] for `O(n)` average k-th-element selection.

## Interview pitfalls & gotchas

- **Off-by-one in partition bounds** — mixing up Lomuto's "return pivot index" with Hoare's "return split" gives wrong recursion ranges and infinite loops.
- **Forgetting the `Θ(n²)` worst case** or its trigger (sorted input + naive pivot). Always mention randomization/median-of-three.
- **`Θ(n²)` on duplicates**: an all-equal array with 2-way Lomuto is worst case; the 3-way fix is the expected answer.
- **Stack overflow** from not recursing on the smaller side.
- **Calling it stable** — it isn't.

## See Also
- [[divide-and-conquer]]
- [[quickselect-and-top-k]]
- [[hybrid-sorts]]
- [[merge-sort]]
- [[heap-sort]]
- [[sorting-algorithms]]
