# Counting Sort

Counting Sort sorts **integer keys drawn from a small range** without a single comparison. It tallies how many times each key occurs, turns those tallies into positions via a prefix sum, then drops each element straight into its final slot. It runs in `Θ(n + k)` for keys in `[0, k)` — **linear when `k = O(n)`** — and, done right, it is **stable**, which is exactly why [[radix-sort]] uses it as a subroutine.

## The idea / invariant

Comparison sorts are bounded below by `Ω(n log n)`. Counting sort sidesteps that bound by not comparing — it uses each key **as an array index**. Three steps:

1. **Count** occurrences of each key into `count[0..k-1]`.
2. **Prefix-sum** `count` so that `count[v]` becomes the number of elements `< v` — i.e. the starting index of value `v` in the output.
3. **Place** each input element at `out[count[key]]`, incrementing `count[key]` as you go.

```
input keys (k=5):  [2 5 3 0 2 3 0 3]   (values 0..4)
count:             [2 0 2 3 0]          two 0s, two 2s, three 3s
prefix (exclusive):[0 2 2 4 7]          value v starts at index prefix[v]
```

## Implementation (stable)

```ts
// Sorts a[] whose keys lie in [0, k). Stable.
function countingSort(a: number[], k: number): number[] {
  const count = new Array(k).fill(0);
  for (const v of a) count[v]++;                 // 1. tally

  for (let v = 1; v < k; v++) count[v] += count[v - 1];
  // now count[v] = number of elements <= v (running total)

  const out = new Array(a.length);
  // 2+3. Iterate input RIGHT-TO-LEFT to preserve input order among equals.
  for (let i = a.length - 1; i >= 0; i--) {
    const v = a[i];
    count[v]--;                                  // slot just before the running total
    out[count[v]] = a[i];
  }
  return out;
}
```

The **right-to-left** placement loop is the crux of stability: because `count[v]` counts down from the *last* position for value `v`, walking the input backward assigns the later-occurring equal keys to later slots — original order is preserved. Iterate left-to-right and you'd reverse equal keys, silently breaking stability (and thus breaking radix sort).

## Dry run

Sort `[2, 5, 3, 0, 2, 3, 0, 3]`, `k = 6`.

- **Count** → `[2, 0, 2, 3, 0, 1]` (two 0s, zero 1s, two 2s, three 3s, zero 4s, one 5).
- **Prefix sum** → `[2, 2, 4, 7, 7, 8]` (running totals).
- **Place (right→left)**: last element is `3` → `count[3]=7`, decrement to 6, `out[6]=3`. Next `0` → `count[0]=2→1`, `out[1]=0`. Next `2` → `count[2]=4→3`, `out[3]=2`. …continuing yields:

```
out = [0, 0, 2, 2, 3, 3, 3, 5]
```

The two `0`s and three `3`s land contiguously; had the input carried satellite data, their original relative order would be intact.

## Complexity

**Time — `Θ(n + k)` (best = average = worst).** One `Θ(n)` pass to count, one `Θ(k)` pass to prefix-sum, one `Θ(n)` pass to place. No input variation changes this. When `k = O(n)` the total is `Θ(n)` — genuinely linear, beating the `Ω(n log n)` comparison lower bound because it isn't a comparison sort.

**Space — `Θ(n + k)`.** The `count` array is `Θ(k)`; the stable version needs a `Θ(n)` output buffer (you cannot place stably in place). So counting sort is **not in-place**.

**Stable — YES**, with right-to-left placement. This property is non-negotiable when it serves as radix sort's per-digit pass.

## The assumption that breaks it

Counting sort's whole premise is that **keys are integers in a bounded, small range**. It falls apart when:

- **`k` is huge relative to `n`.** Sorting `n = 1000` values in range `k = 10^9` allocates a billion-slot array — `Θ(k)` space and time swamp everything. Rule of thumb: only worthwhile when `k = O(n)` (or a small multiple).
- **Keys are floats / continuous / arbitrary strings.** There is no finite index space to count into. Use [[bucket-sort]] (partition a continuous range) or map keys to a bounded integer domain first.
- **Negative keys** — shift by `-min` (index `v - min`) so indices stay non-negative.

## Variants & trade-offs

- **Non-stable, in-`count`-only variant**: if you only need sorted keys without satellite data, skip the output buffer and re-emit `count[v]` copies of each `v`. Loses stability and can't carry associated records, but uses only `Θ(k)` extra space.
- vs [[radix-sort]]: radix sort *is* repeated counting sort, one digit at a time, to extend the technique to large key ranges while keeping linear-ish time.
- The prefix-sum step is a textbook use of [[prefix-sums]] — cumulative counts become positions.

## When to use / when not

- **Use** for large `n` over a small integer domain: ages, byte values, small enum categories, exam scores 0–100, or as the digit-sort inside radix sort.
- **Don't** use for wide/sparse key ranges, floating-point, or comparison-only keys — the `Θ(k)` cost is fatal.

## Interview pitfalls & gotchas

- **Placing left-to-right and calling it stable** — the single most common bug; walk the input backward.
- **Off-by-one in the prefix sum** — decide up front whether `count[v]` is "elements `< v`" (exclusive, place then increment) or "elements `<= v`" (inclusive, decrement then place). The code above uses the inclusive form and decrements first; mixing conventions overwrites slots.
- **`k` chosen as `max` instead of `max + 1`** — the array needs one slot per value including the maximum.
- **Ignoring negatives** — forgetting the `-min` offset throws index errors.
- **Assuming it beats `n log n` unconditionally** — only when `k` stays small; otherwise it's worse.

## See Also
- [[radix-sort]]
- [[prefix-sums]]
- [[bucket-sort]]
- [[sorting-algorithms]]
- [[asymptotic-notation]]
