# Selection Sort

Selection sort builds the sorted output one element at a time: on each pass it **scans the entire unsorted suffix to find the minimum**, then swaps that minimum into the first unsorted slot. It is `Θ(n²)` in *every* case, not stable (in the usual implementation), but performs exactly `n−1` swaps — the fewest writes of any comparison sort, which is its single redeeming feature.

## The Idea / Invariant

Split the array into a **sorted prefix** (initially empty) and an **unsorted suffix** (initially everything). Each pass finds the smallest element of the suffix and swaps it to the front of the suffix, extending the prefix by one.

**Invariant:** after `i` passes, `a[0..i-1]` holds the `i` smallest elements in final sorted order, and every element there is ≤ every element in the suffix.

```
prefix (sorted, final)      suffix (unsorted)
[  ][ 64  25  12  22  11 ]   find min=11, swap with 64
[11][ 25  12  22  64      ]  find min=12, swap with 25
[11 12][ 25  22  64       ]  ...
```

The defining property: the scan for the minimum is **unconditional** — it always inspects every remaining element regardless of how sorted the data already is. That is why there is no "best case".

## Implementation

```ts
function selectionSort(a: number[]): number[] {
  const n = a.length;
  for (let i = 0; i < n - 1; i++) {
    let min = i;                       // index of smallest in suffix a[i..n-1]
    for (let j = i + 1; j < n; j++) {  // ALWAYS scans the whole suffix
      if (a[j] < a[min]) min = j;      // '<' → keeps the FIRST of equal minima
    }
    if (min !== i) {                   // one swap per pass, at most n-1 total
      [a[i], a[min]] = [a[min], a[i]];
    }
  }
  return a;
}
```

The inner loop only *reads* and updates an index; the single swap happens once per outer pass. Total swaps ≤ `n−1` — dramatically fewer than bubble sort's `Θ(n²)`.

## Dry Run / Trace

Input `[64, 25, 12, 22, 11]`. `|` marks the prefix/suffix boundary.

| pass `i` | array before | scan finds min | swap | array after |
|---|---|---|---|---|
| 0 | `\| 64 25 12 22 11` | `11` @4 | swap a[0]↔a[4] | `11 \| 25 12 22 64` |
| 1 | `11 \| 25 12 22 64` | `12` @2 | swap a[1]↔a[2] | `11 12 \| 25 22 64` |
| 2 | `11 12 \| 25 22 64` | `22` @3 | swap a[2]↔a[3] | `11 12 22 \| 25 64` |
| 3 | `11 12 22 \| 25 64` | `25` @3 | `min===i`, no swap | `11 12 22 25 \| 64` |

Four passes, **three actual swaps**, `64` never moved after being displaced. Final `[11, 12, 22, 25, 64]`.

Note pass 0 scanned all 4 suffix elements *even though* nothing after was smaller than 25/12 individually — the scan is blind to structure.

## Complexity

- **Time — `Θ(n²)` in all cases (best = average = worst).** The comparison count is fixed by the loop shape, independent of the data: `(n−1)+(n−2)+…+1 = n(n−1)/2 = Θ(n²)`. Feeding it an already-sorted array does **not** help — it still runs every comparison. This is the sharp contrast with [[bubble-sort]]/[[insertion-sort]], which have an `O(n)` best case.
- **Swaps — exactly `n−1` in the worst case, `0` if already sorted** (`min === i` every pass). This `O(n)` write count is the algorithm's one virtue.
- **Space — `O(1)`.** In-place, one temp for the swap.
- **Stable — no** (typical index-swap implementation, see below).
- **In-place — yes.**

## Stability: Why the Tie-Swap Breaks It

The swap `a[i] ↔ a[min]` can **hurdle** an element over an equal-keyed one. Sort by the numeric key with letters as tie-tags:

```
input:  (5a) (3)  (5b) (2)
pass 0: min is (2)@3 → swap a[0]↔a[3]
result: (2)  (3)  (5b) (5a)   ← 5a and 5b swapped order!
```

`5a` started before `5b`, but the pass-0 swap threw `5a` to the back, landing it *after* `5b`. Equal keys got reordered → **unstable**. You can make selection sort stable by *shifting* the minimum into place instead of swapping (like insertion), but that reintroduces `Θ(n²)` writes and forfeits its only advantage.

## Variants & Trade-offs

- **Bidirectional / double selection** finds min *and* max in one pass, filling both ends — halves the passes, same `Θ(n²)`.
- **[[heap-sort]]** is the "smart" selection sort: it also repeatedly selects the extremum, but from a [[heaps|heap]] in `Θ(log n)` instead of an `Θ(n)` linear scan — turning the whole thing into `Θ(n log n)`.
- Versus [[bubble-sort]]: same time class, but selection sort does `O(n)` swaps vs bubble's `O(n²)`, so it moves data far less.

## When to Use / When Not

**Use** it only when **writes are the expensive operation** and reads are cheap: flashing to EEPROM/flash memory (limited write-erase cycles), or any medium where each store is costly. Its guaranteed `≤ n−1` swaps beat every other sort on write count. **Do not use** it otherwise — the `Θ(n²)` comparison count and lack of an adaptive best case make it slower than insertion sort on essentially all normal inputs.

## Interview Pitfalls & Gotchas

- **Claiming an `O(n)` best case** — there is none; the scan is unconditional. This is the most common selection-sort mistake.
- **Assuming it's stable** — the tie-swap reorders equal keys; do not rely on it for secondary-key ordering.
- **Skipping the `min !== i` guard** — harmless for correctness but wastes writes (the whole point of the algorithm is minimal writes).
- **Confusing it with insertion sort** — selection *selects the global min of the suffix*; insertion *inserts the next element into the sorted prefix*. Different invariants, different best cases.
- **Off-by-one in the outer loop** — `i` runs to `n−2` (the last element is trivially in place once the rest are sorted).

## See Also

- [[insertion-sort]]
- [[bubble-sort]]
- [[heap-sort]]
- [[sorting-algorithms]]
- [[big-o-vs-reality|Big-O vs Reality]]
