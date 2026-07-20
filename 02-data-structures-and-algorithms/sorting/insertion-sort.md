# Insertion Sort

Insertion sort grows a sorted prefix one element at a time: it takes the next unsorted element and **shifts** the larger sorted elements right until it finds the slot where the new element belongs, then drops it in. It is `Θ(n²)` in the worst case but `Θ(n)` on nearly-sorted input, stable, in-place, and **online** — and its tiny constant factors make it the fallback that Timsort and introsort use for small subarrays.

## The Idea / Invariant

Think of sorting a hand of playing cards: you keep the cards in your hand ordered, and each new card is slid left past the bigger ones into its place.

**Invariant:** after processing index `i`, the prefix `a[0..i]` is sorted (though not necessarily in final positions, since later elements may still be smaller). Each step preserves this by inserting `a[i]` into the already-sorted `a[0..i-1]`.

```
sorted prefix | key to insert
[2 5 8] | 4      shift 5,8 right, drop 4 →
[2 4 5 8] | ...
```

The insertion is done by **shifting**, not swapping: cache the key, slide bigger elements one slot right, then write the key once into the gap. Fewer writes than the swap-based framing.

## Implementation

```ts
function insertionSort(a: number[]): number[] {
  for (let i = 1; i < a.length; i++) {
    const key = a[i];                  // element to place into the sorted prefix
    let j = i - 1;
    // slide every element strictly greater than key one slot to the right
    while (j >= 0 && a[j] > key) {      // '>' (not '>=') stops at equals → STABLE
      a[j + 1] = a[j];
      j--;
    }
    a[j + 1] = key;                    // drop key into the opened gap
  }
  return a;
}
```

The guard `a[j] > key` (strict) is what makes it **stable**: the scan halts as soon as it meets an element equal to `key`, so `key` lands *after* its equals, preserving input order. The inner `while` does the work; on nearly-sorted data it almost never executes.

## Dry Run / Trace

Input `[5, 2, 4, 6, 1]`. `key` is the element being inserted; the box shows the sorted prefix.

| i | key | before | shifts | after |
|---|---|---|---|---|
| 1 | 2 | `[5] 2 4 6 1` | 5→right | `[2 5] 4 6 1` |
| 2 | 4 | `[2 5] 4 6 1` | 5→right | `[2 4 5] 6 1` |
| 3 | 6 | `[2 4 5] 6 1` | none (6>5) | `[2 4 5 6] 1` |
| 4 | 1 | `[2 4 5 6] 1` | 6,5,4,2→right | `[1 2 4 5 6]` |

Step `i=3` is the cheap case: `6 > 5` so the `while` never runs — one comparison, zero shifts. Step `i=4` is the expensive case: `1` is smaller than everything, so all four prefix elements shift. Final `[1, 2, 4, 5, 6]`.

## Complexity

Let `n` be the length; the cost is dominated by inner-loop shifts, which equal the number of **inversions** in the input.

- **Worst case — `Θ(n²)`.** Reverse-sorted input forces the `while` to shift the entire prefix each step: `1+2+…+(n−1) = n(n−1)/2 = Θ(n²)` shifts and comparisons.
- **Average case — `Θ(n²)`.** A random permutation has `n(n−1)/4` expected inversions → still `Θ(n²)`.
- **Best case — `Θ(n)`.** An already-sorted (or nearly-sorted) array: each `while` guard fails immediately, so each element costs one comparison → `n−1` comparisons, zero shifts. More precisely the running time is `Θ(n + inversions)`, making it **adaptive** — cost scales with how out-of-order the input actually is.
- **Space — `O(1)`.** In-place; one `key` temporary.
- **Stable — yes**, via the strict `>` guard (equal elements are never shifted past each other).
- **In-place — yes.**
- **Online — yes.** It can accept elements one at a time and keep the processed prefix sorted, so it sorts a **stream** without seeing the whole input first.

## Why Timsort/Introsort Fall Back to It

For small `n` (typically 16–32), insertion sort **beats** `Θ(n log n)` sorts in wall-clock time. Reasons: no recursion overhead, sequential memory access (excellent [[07-performance-engineering/data-locality|data locality]]), predictable branches, and the `Θ(n²)` term is negligible when `n` is a small constant. So [[merge-sort]]/[[quick-sort]] recursion bottoms out into insertion sort, and Timsort builds its initial "runs" with a binary-insertion variant. See [[hybrid-sorts]] for how the cutoff is chosen.

## Variants & Trade-offs

- **Binary insertion sort** uses [[binary-search]] to find the insertion point in `O(log i)` comparisons instead of `O(i)` — cuts *comparisons* to `Θ(n log n)`, but *shifts* remain `Θ(n²)`, so total time is unchanged; useful when comparisons are far costlier than moves.
- **[[shell-sort]]** generalizes insertion sort to gapped passes, letting elements jump far early and breaking the `Θ(n²)` inversion wall.
- Versus [[selection-sort]]: both `Θ(n²)`, but insertion sort is adaptive (`O(n)` best) and stable; selection sort is neither (but does fewer writes).

## When to Use / When Not

**Use** it for small arrays, nearly-sorted data, streaming/online input, and as the base case inside larger sorts. **Do not use** it as a general-purpose sort for large random arrays — the `Θ(n²)` shift count dominates; reach for [[merge-sort]]/[[quick-sort]].

## Interview Pitfalls & Gotchas

- **Using `>=` instead of `>`** in the shift guard — shifts equal elements too, making the sort **unstable** and doing needless work.
- **Off-by-one at `a[j + 1] = key`** — after the loop, `j` points *left* of the gap, so the key goes at `j+1`; writing `a[j]` corrupts the array.
- **Forgetting `j >= 0`** in the guard — walking off the left end when the key is the new minimum (JS returns `undefined`, silently breaking the compare).
- **Claiming `Θ(n²)` always** — it is adaptive; on sorted/nearly-sorted input it is `Θ(n)`, which is exactly why it is used in hybrids.
- **Confusing "comparisons" with "moves"** — binary insertion reduces comparisons but not moves; the asymptotic time is set by the moves.

## See Also

- [[shell-sort]]
- [[selection-sort]]
- [[bubble-sort]]
- [[merge-sort]]
- [[hybrid-sorts]]
- [[sorting-algorithms]]
