# Merge Sort

Merge Sort is the archetypal divide-and-conquer sort: split the array in half, sort each half recursively, then **merge** the two sorted halves into one. Reach for it when you need a **guaranteed** `Θ(n log n)` with **stable** ordering, when sorting **linked lists**, or when data is too big for RAM (external sort).

## The idea / invariant

The whole algorithm rests on one fact: **merging two already-sorted sequences into one sorted sequence takes linear time** with a single pass of two pointers. Recursion produces those sorted halves; merge combines them. The invariant during a merge is: everything already written to the output is sorted and `≤` every element still unconsumed in both inputs.

```
[38 27 43 3 9 82 10]
     split ->  [38 27 43]      [3 9 82 10]
     split ->  [38][27 43]     [3 9][82 10]
     merge <-  [27 38 43]      [3 9 10 82]
     merge <-  [3 9 10 27 38 43 82]
```

## Implementation (top-down, stable)

```ts
function mergeSort(a: number[]): number[] {
  if (a.length <= 1) return a;               // base case: 0/1 elements are sorted
  const mid = a.length >> 1;                 // floor(n/2), overflow-safe here
  const left = mergeSort(a.slice(0, mid));   // sort left half
  const right = mergeSort(a.slice(mid));     // sort right half
  return merge(left, right);
}

function merge(left: number[], right: number[]): number[] {
  const out: number[] = [];
  let i = 0, j = 0;
  while (i < left.length && j < right.length) {
    // "<=" (take LEFT on ties) is what makes the sort STABLE:
    // equal keys from the earlier half are emitted first.
    if (left[i] <= right[j]) out.push(left[i++]);
    else out.push(right[j++]);
  }
  while (i < left.length) out.push(left[i++]);   // drain leftovers
  while (j < right.length) out.push(right[j++]);
  return out;
}
```

The single most important detail is `left[i] <= right[j]`. Using `<` (taking the right element on a tie) would swap the relative order of equal keys and **break stability**.

## Dry run — merging two runs

Merge `left = [27, 38, 43]`, `right = [3, 9, 10, 82]`:

| step | compare | emit | out |
|------|---------|------|-----|
| 1 | 27 vs 3 | 3 | `[3]` |
| 2 | 27 vs 9 | 9 | `[3,9]` |
| 3 | 27 vs 10 | 10 | `[3,9,10]` |
| 4 | 27 vs 82 | 27 | `[3,9,10,27]` |
| 5 | 38 vs 82 | 38 | `[3,9,10,27,38]` |
| 6 | 43 vs 82 | 43 | `[3,9,10,27,38,43]` |
| — | right exhausted | drain 82 | `[3,9,10,27,38,43,82]` |

Seven elements, six comparisons then a drain — linear in the combined length.

## Complexity

**Time — `Θ(n log n)` in best, average, and worst case.** Set up the recurrence: sorting `n` elements does two subproblems of size `n/2` plus a merge that touches every element once:

```
T(n) = 2·T(n/2) + Θ(n),   T(1) = Θ(1)
```

By the [[recurrence-relations|Master Theorem]] this is case 2 (`a = b = 2`, `f(n) = Θ(n) = Θ(n^{log_b a})`), giving `T(n) = Θ(n log n)`. Intuitively there are `log₂ n` levels of recursion, and each level does `Θ(n)` total merging work regardless of the input, so **there is no lucky or unlucky input** — every case is `n log n`. That worst-case guarantee is Merge Sort's headline advantage over [[quick-sort]].

**Space — `Θ(n)` auxiliary.** The `merge` needs a scratch buffer the size of the run being merged; the classic implementation allocates `Θ(n)`. (Recursion adds `Θ(log n)` stack.) True **in-place merge** exists but is either `O(n log²n)` or extremely intricate with large constants, so in practice merge sort is *not* considered in-place — this is its main drawback versus [[heap-sort]] and quicksort.

**Stable — yes**, given the `<=` tie rule.

## Top-down vs bottom-up

- **Top-down** (above) recurses then merges. Simple, but the recursion allocates.
- **Bottom-up** skips recursion: treat the array as `n` runs of length 1, then merge adjacent runs of width 1, 2, 4, 8, … doubling each pass. `⌈log₂ n⌉` passes, same `Θ(n log n)`. It is the natural form for **external merge sort** (merge sorted chunks streamed from disk) and avoids stack depth.

## When to use / when not

- **Linked lists**: merge sort shines. Splitting is `O(n)` via a [[fast-slow-pointers|fast/slow pointer]], merging just relinks nodes, and crucially it needs **no random access and no extra array** — you get `Θ(n log n)` stable sort with `O(1)` extra data space (only `O(log n)` stack). This is why library list-sorts are merge sorts. See [[linked-lists]].
- **External / big-data sorting**: when data exceeds memory, sort chunks that fit, write them out, then k-way merge the sorted runs. Sequential access only.
- **Stability required**: sorting records by a secondary key while preserving primary order.
- **Avoid** when memory is tight and data is an array — the `Θ(n)` buffer hurts, and quicksort is usually faster in wall-clock due to better cache locality and lower constants.

## Interview pitfalls & gotchas

- **Breaking stability** with `<` instead of `<=` in the merge comparison. Interviewers probe this exact line.
- **Off-by-one on `mid` / slice bounds** — `a.slice(0, mid)` and `a.slice(mid)` must partition without overlap or gaps; an empty right half causes infinite recursion.
- **Forgetting to drain** the non-exhausted input after the main loop drops elements silently.
- **Claiming it's in-place.** It is not, for arrays. Say `Θ(n)` auxiliary space plainly.
- **`mid` overflow** in fixed-width-int languages: use `low + ((high - low) >> 1)` rather than `(low + high) / 2`. Irrelevant to JS numbers but a classic C/Java bug.
- **Excess allocation**: `slice` copies each level. Production versions merge into a single reused scratch buffer with index ranges to cut allocation and GC pressure.

## See Also
- [[divide-and-conquer]]
- [[recurrence-relations]]
- [[linked-lists]]
- [[quick-sort]]
- [[hybrid-sorts]]
- [[sorting-algorithms]]
