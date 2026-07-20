# Bucket Sort

Bucket Sort scatters elements into a fixed number of **buckets** by value range, sorts each bucket (usually with [[insertion-sort]]), then concatenates the buckets in order. When the input is **uniformly distributed** over a known range, each bucket stays tiny and the whole thing runs in **expected `Θ(n + k)`** — linear. When the distribution is skewed, it degrades to `Θ(n²)`.

## The idea / invariant

If `n` values are spread uniformly across `[0, 1)` and you create `n` buckets — bucket `i` covering `[i/n, (i+1)/n)` — then *on average* each bucket receives exactly one element. Sorting one-element buckets is free; concatenating buckets `0, 1, 2, …` in order yields a sorted array because every element in bucket `i` is `< ` every element in bucket `i+1` by construction. The buckets impose a coarse order; the per-bucket sort refines it.

```
input in [0,1):  [.78 .17 .39 .26 .72 .94 .21 .12 .23]
buckets (x10):   0:[]  1:[.17 .12] 2:[.26 .21 .23] 3:[.39] 7:[.78 .72] 9:[.94]
sort each:          [.12 .17]      [.21 .23 .26]
concat:          [.12 .17 .21 .23 .26 .39 .72 .78 .94]
```

## Implementation (floats in `[0, 1)`)

```ts
function bucketSort(a: number[]): number[] {
  const n = a.length;
  if (n <= 1) return a;
  const buckets: number[][] = Array.from({ length: n }, () => []);

  // scatter: value v (in [0,1)) -> bucket floor(v * n)
  for (const v of a) buckets[Math.floor(v * n)].push(v);

  const out: number[] = [];
  for (const bucket of buckets) {
    insertionSort(bucket);   // cheap: buckets are expected to be tiny
    out.push(...bucket);     // gather in bucket order
  }
  return out;
}

function insertionSort(b: number[]): void {
  for (let i = 1; i < b.length; i++) {
    const key = b[i];
    let j = i - 1;
    while (j >= 0 && b[j] > key) { b[j + 1] = b[j]; j--; }
    b[j + 1] = key;
  }
}
```

For a general range `[min, max)`, index each value with `Math.floor((v - min) / (max - min) * n)`.

## Dry run

Sort `[0.42, 0.17, 0.91, 0.13, 0.48]`, `n = 5`, bucket index `= floor(v*5)`:

| value | v*5 | bucket |
|-------|-----|--------|
| 0.42 | 2.1 | 2 |
| 0.17 | 0.85 | 0 |
| 0.91 | 4.55 | 4 |
| 0.13 | 0.65 | 0 |
| 0.48 | 2.4 | 2 |

Buckets: `0:[0.17, 0.13]`, `2:[0.42, 0.48]`, `4:[0.91]`, others empty. Insertion-sort each → `0:[0.13, 0.17]`, `2:[0.42, 0.48]`. Concatenate buckets 0..4: `[0.13, 0.17, 0.42, 0.48, 0.91]`. Sorted.

## Complexity

**Time.**
- **Expected / average — `Θ(n + k)`** where `k` is the bucket count (typically `k = n`). Scatter is `Θ(n)`; gather/concatenation is `Θ(n + k)`. The subtle part is the per-bucket sort. Insertion sort on a bucket of size `n_i` costs `O(n_i²)`. Total sorting cost is `Σ O(n_i²)`. Under the **uniform-distribution assumption**, each `n_i` is a binomial random variable with mean 1, and `E[Σ n_i²] = Θ(n)`. So the expected total is `Θ(n)`. This is an **average-case bound that leans entirely on the input distribution**, unlike the distribution-free guarantees of merge/heap sort.
- **Worst — `Θ(n²)`.** If the distribution is adversarial or skewed and **all `n` elements fall in one bucket**, that bucket's insertion sort is `O(n²)` and bucket sort degenerates to a single insertion sort. Non-uniform real data (clusters, power-law) pushes toward this.
- **Best — `Θ(n + k)`** (already-uniform, one element per bucket).

**Space — `Θ(n + k)`** for the buckets and their contents. **Not in-place.**

**Stable — yes, if** the per-bucket sort is stable and scatter preserves input order (pushing in input order into each bucket does). With stable insertion sort, bucket sort is stable.

## The uniform-distribution assumption

The `Θ(n)` expectation is *only* valid when values are (roughly) uniform across the chosen range. This is the algorithm's defining caveat: it is a **distribution-dependent** sort. Feed it clustered data and buckets fill unevenly, the `Σ n_i²` term explodes, and you lose the linear time. If you don't know the distribution, don't reach for bucket sort — use a comparison sort that doesn't care.

## Relation to counting & radix sort

- **[[counting-sort]]** is bucket sort's degenerate case where each bucket holds a single key value — no per-bucket sorting needed, just counts. Counting sort assumes discrete integer keys; bucket sort assumes a continuous range partitioned into intervals.
- **[[radix-sort]]** can be seen as repeated bucketing digit-by-digit. All three are **non-comparison, distribution-exploiting** sorts that break the `Ω(n log n)` barrier under the right key assumptions.

## When to use / when not

- **Use** for **uniformly distributed floating-point** data over a known range (sensor readings normalized to `[0,1)`, random fractions, hash-spread values), or when data is already known to be evenly spread.
- **Don't** use for unknown or skewed distributions (worst case bites), for data without a natural numeric range, or when `O(1)` extra space is required.

## Interview pitfalls & gotchas

- **Assuming `O(n)` regardless of input** — it's `O(n)` *expected under uniformity*; worst case is `O(n²)`. State the assumption explicitly.
- **Wrong bucket index / boundary** — `Math.floor(v * n)` for `v` exactly `1.0` yields index `n` (out of range); the domain must be `[0, 1)` or you must clamp.
- **Too few or too many buckets** — with `k ≪ n` buckets get large and per-bucket sort dominates; with `k ≫ n` you waste memory on empty buckets.
- **Choosing an `O(n²)` per-bucket sort blindly** — fine for tiny buckets (insertion sort has low constants), disastrous if buckets can grow large; some implementations recurse with bucket sort or use a comparison sort as a hedge.
- **Forgetting stability requirements** if you rely on it.

## See Also
- [[insertion-sort]]
- [[counting-sort]]
- [[radix-sort]]
- [[sorting-algorithms]]
- [[quick-sort]]
