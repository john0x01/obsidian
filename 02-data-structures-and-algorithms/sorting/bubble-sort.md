# Bubble Sort

Bubble sort repeatedly walks the array, swapping any two **adjacent** elements that are out of order, so that on each pass the largest unsorted element "bubbles up" to its final position at the end. It is `Θ(n²)`, stable, and in-place — and it is essentially **pedagogical only**: you learn it to understand swaps, passes, and invariants, then you reach for [[insertion-sort]] or a real sort instead.

## The Idea / Invariant

One pass compares each neighbouring pair `(a[j], a[j+1])` left to right and swaps when `a[j] > a[j+1]`. The largest element can never be swapped *down*, so after pass 1 the maximum sits at index `n−1`; after pass 2 the second-largest sits at `n−2`, and so on.

**Invariant:** after `k` passes, the last `k` elements are the `k` largest, in sorted order. So each pass can stop `k` elements earlier — the sorted suffix grows by one every pass.

```
sorted suffix →              []
pass 1:  [5 1 4 2 8]   bubbles 8 to the end → [1 4 2 5 | 8]
pass 2:  [1 4 2 5]     bubbles 5            → [1 2 4 | 5 8]
```

The **early-exit flag** is the one non-trivial idea: if an entire pass makes zero swaps, the array is already sorted and we stop. That is what turns the best case from `Θ(n²)` into `Θ(n)`.

## Implementation

```ts
function bubbleSort(a: number[]): number[] {
  const n = a.length;
  for (let i = 0; i < n - 1; i++) {
    let swapped = false;                 // early-exit sentinel
    // after i passes, last i elements are already in place → stop at n-1-i
    for (let j = 0; j < n - 1 - i; j++) {
      if (a[j] > a[j + 1]) {             // strict '>' keeps equal keys in order → STABLE
        [a[j], a[j + 1]] = [a[j + 1], a[j]];
        swapped = true;
      }
    }
    if (!swapped) break;                 // no swaps this pass ⇒ fully sorted
  }
  return a;
}
```

Two details carry the algorithm's whole character: the inner bound `n - 1 - i` (never re-scan the sorted suffix) and the `swapped` flag (bail early on sorted input). Using `>` rather than `>=` is what makes it **stable** — equal elements are never swapped past each other.

## Dry Run / Trace

Input `[5, 1, 4, 2, 8]`. Each row is the array after processing one adjacent pair; `^` marks the pair compared.

**Pass 1** (`i=0`, scan `j=0..3`):

| step | array | pair | action |
|---|---|---|---|
| start | `5 1 4 2 8` | — | — |
| j=0 | `1 5 4 2 8` | `5>1` | swap |
| j=1 | `1 4 5 2 8` | `5>4` | swap |
| j=2 | `1 4 2 5 8` | `5>2` | swap |
| j=3 | `1 4 2 5 8` | `5<8` | no swap |

`8` is now locked at the end. `swapped = true`.

**Pass 2** (`i=1`, scan `j=0..2`): `1 4 2 5` → j=0 `1<4` no; j=1 `4>2` swap → `1 2 4 5`; j=2 `4<5` no. Result `[1 2 4 5 | 8]`, `swapped = true`.

**Pass 3** (`i=2`, scan `j=0..1`): `1 2 4` → `1<2` no, `2<4` no. **Zero swaps** → `swapped = false` → **break**. Final `[1, 2, 4, 5, 8]`.

The early exit saved passes 4 and 5.

## Complexity

Let `n` be the length.

- **Worst case — `Θ(n²)` time.** A reverse-sorted array forces a swap at every comparison. Total comparisons `= (n−1)+(n−2)+…+1 = n(n−1)/2 = Θ(n²)`, and just as many swaps. The early-exit flag never trips.
- **Average case — `Θ(n²)` time.** A random permutation still needs a constant fraction of the `Θ(n²)` comparisons and `Θ(n²)` swaps (expected number of inversions is `n(n−1)/4`).
- **Best case — `Θ(n)` time**, but *only with the flag*: an already-sorted array does one full pass of `n−1` comparisons, makes zero swaps, and breaks. Without the flag it degrades to `Θ(n²)` even on sorted input.
- **Space — `O(1)`.** Sorts within the array; swaps use one temp slot.
- **Stable — yes.** Equal elements are compared with `>` and never swapped, so their relative order is preserved.
- **In-place — yes.** No auxiliary array.

## Variants & Trade-offs

- **Cocktail (bidirectional) sort** alternates left-to-right and right-to-left passes, so small elements ("turtles") near the end migrate faster. Same `Θ(n²)`, marginally fewer passes.
- **Comb sort** compares elements a shrinking *gap* apart (like [[shell-sort]] applied to bubble sort), killing turtles and reaching near-`Θ(n log n)` empirically.
- Versus [[insertion-sort]]: both are `Θ(n²)` and stable and `O(n)` on sorted input, but insertion sort does far fewer *writes* and has better constants — it is strictly the better choice.

## When to Use / When Not

**Use** essentially never in production. Its only real roles: teaching swap-based sorting and invariants, and detecting "is this already sorted?" in one `O(n)` pass via the flag. **Do not use** it for real workloads — even at small `n`, insertion sort wins; at large `n`, its `Θ(n²)` swap count is catastrophic.

## Interview Pitfalls & Gotchas

- **Omitting the `swapped` flag** — then you cannot claim the `O(n)` best case; sorted input still costs `Θ(n²)`.
- **Wrong inner bound** — using `j < n - 1` instead of `n - 1 - i` re-scans the already-sorted suffix; correct output but wasted `Θ(n²)/2` comparisons.
- **Using `>=` instead of `>`** — swaps equal elements, silently making the sort **unstable**.
- **Claiming it's "the simplest so it's fine for small n"** — insertion sort is just as simple and materially faster; interviewers expect you to know that.
- **Confusing passes with the answer** — the array can be fully sorted before the last element is "bubbled"; only the flag detects that.

## See Also

- [[insertion-sort]]
- [[selection-sort]]
- [[shell-sort]]
- [[sorting-algorithms]]
- [[big-o-vs-reality|Big-O vs Reality]]
