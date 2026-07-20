# Shell Sort

Shell sort is **insertion sort run over diminishing gaps**: instead of only comparing adjacent elements, it first sorts elements that are a large gap `g` apart, then shrinks `g` down to 1. The early wide-gap passes let elements leap across the array toward their final region, so by the time `g = 1` (ordinary insertion sort) the array is nearly sorted and the final pass is cheap. It is in-place, **not stable**, and its complexity is entirely determined by the gap sequence — from `Θ(n²)` for Shell's original gaps down to roughly `Θ(n^1.3)` for good ones.

## The Idea / Invariant

Plain [[insertion-sort]] moves each element at most one slot per shift, so an element that is far from home needs many shifts — the cost is the number of *inversions*, which can be `Θ(n²)`. Shell sort's insight: sort **`g`-sorted subsequences** first. An array is "`g`-sorted" when every element is ≤ the element `g` positions ahead. A single misplaced small element at the end can hop `g` positions in one move, eliminating many inversions at once.

**Invariant:** after the pass with gap `g`, the array is `g`-sorted; and crucially a `g`-sorted array stays `g`-sorted after any later smaller-gap pass. So inversions are removed coarsely first, finely last, and the final `g = 1` pass only cleans up the few short-range disorders that remain.

```
gap=4:  a[0],a[4],a[8]  form one interleaved subsequence — insertion-sort it
        a[1],a[5],a[9]  another …
gap=1:  ordinary insertion sort, but now nearly sorted → near O(n)
```

It is the **generalization of insertion sort**: gap `1` *is* insertion sort. Shell sort trades a few extra passes for drastically fewer total shifts.

## Implementation

```ts
function shellSort(a: number[]): number[] {
  const n = a.length;
  // Ciura's empirically-strong gaps, largest that fits first
  const gaps = [701, 301, 132, 57, 23, 10, 4, 1];
  for (const gap of gaps) {
    if (gap >= n) continue;
    // gapped insertion sort: same logic as insertion sort, step = gap not 1
    for (let i = gap; i < n; i++) {
      const key = a[i];
      let j = i;
      while (j >= gap && a[j - gap] > key) { // shift by 'gap', not 1
        a[j] = a[j - gap];
        j -= gap;
      }
      a[j] = key;
    }
  }
  return a;
}
```

Structurally this is [[insertion-sort]] with the constant `1` replaced by `gap`; the outer loop just repeats it for shrinking gaps. The subsequences are processed interleaved (the `i` loop walks all of them together), which is equivalent to sorting each independently.

## Dry Run / Trace — One Gap Pass

Input `[9, 8, 3, 7, 5, 6, 4, 1]`, `n = 8`. Ciura gaps that fit: `4, 1`.

**Gap = 4.** The gap-4 subsequences are `(a0,a4)=(9,5)`, `(a1,a5)=(8,6)`, `(a2,a6)=(3,4)`, `(a3,a7)=(7,1)`. Gapped insertion sort each:

| i | key | compares a[i-4] | action | array |
|---|---|---|---|---|
| 4 | 5 | `9 > 5` | shift 9→idx4, place 5@0 | `5 8 3 7 9 6 4 1` |
| 5 | 6 | `8 > 6` | shift 8→idx5, place 6@1 | `5 6 3 7 9 8 4 1` |
| 6 | 4 | `3 < 4` | no shift | `5 6 3 7 9 8 4 1` |
| 7 | 1 | `7 > 1` | shift 7→idx7, place 1@3 | `5 6 3 1 9 8 4 7` |

After the gap-4 pass the array is `[5, 6, 3, 1, 9, 8, 4, 7]` — notice `9` and `8` already migrated toward the right, and `1` jumped 4 slots left in a single move.

**Gap = 1** (ordinary insertion sort) then finishes on this now-less-disordered array, producing `[1, 3, 4, 5, 6, 7, 8, 9]` with far fewer shifts than insertion sort on the raw input would have needed.

## Complexity

The gap sequence dictates everything; the analysis is famously subtle.

- **Shell's original gaps** `n/2, n/4, …, 1` — **worst case `Θ(n²)`.** These gaps share factors of 2, so odd/even positions never interact until the last pass, leaving pathological inputs.
- **Hibbard `2^k − 1`** — `Θ(n^1.5)` worst case.
- **Sedgewick** — `Θ(n^4/3)` worst case.
- **Ciura's sequence** (used above) — no proven closed form, but empirically the **fastest known**, behaving like roughly `Θ(n^1.3)` on practical sizes.
- **Best case — `Θ(n log n)`** with a good sequence (already-sorted input: each gapped pass does only its `n/g` guard-checks, summed over the sequence).
- **Space — `O(1)`.** Gapped insertion needs only the `key` temporary — **in-place.**
- **Stable — no.** Gapped moves jump equal elements over one another across the subsequences, reordering equal keys.

The takeaway: shell sort is a rare `Θ(n²)`-family algorithm that is *sub-quadratic* with the right gaps, sitting between insertion sort and the `Θ(n log n)` sorts.

## Variants & Trade-offs

- The variant *is* the gap sequence; picking Ciura/Sedgewick over `n/2` is the single biggest lever.
- Versus [[insertion-sort]]: strictly better on large-`n` random data (fewer total shifts), but loses insertion sort's stability and its clean `O(n + inversions)` adaptivity story.
- Versus [[quick-sort]]/[[heap-sort]]: shell sort is slower asymptotically but is **non-recursive, `O(1)` space, and cache-friendly**, with a tiny code footprint.

## When to Use / When Not

**Use** it in constrained environments — embedded/firmware, kernels (Linux once used it), or bootloaders — where you want better-than-`Θ(n²)` performance without recursion, without an `O(n)` merge buffer, and with a handful of lines of code. **Do not use** it when you need stability, or when you can afford a full `Θ(n log n)` sort for large `n`.

## Interview Pitfalls & Gotchas

- **Using `n/2` halving gaps and expecting speed** — that is the `Θ(n²)` sequence; the gap choice is the whole game.
- **Assuming stability** — shell sort is not stable; do not rely on secondary-key order.
- **Miscoding the gapped loop** — `j -= gap` and the guard `j >= gap` must both use the gap; mixing in a `1` breaks the interleaving.
- **Claiming a tidy Big-O** — there is no single bound; you must state *which gap sequence*.
- **Gaps not shrinking to 1** — if the sequence never reaches gap 1 the array is left only partially sorted; the final pass *must* be `g = 1`.

## See Also

- [[insertion-sort]]
- [[bubble-sort]]
- [[quick-sort]]
- [[sorting-algorithms]]
- [[recurrence-relations]]
- [[big-o-vs-reality|Big-O vs Reality]]
