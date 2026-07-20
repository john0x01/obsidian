# Jump Search

Jump search (block search) finds a target in a **sorted** array by leaping ahead in fixed-size blocks until it overshoots the target, then scanning linearly *backward* within the last block. It sits between linear and binary search: worse than binary's `O(log n)`, but it touches memory in long forward strides and only ever backs up once, which makes it attractive on storage where a big backward jump is cheaper than binary search's constant random jumps.

## The Idea

Pick a block size `s`. Probe indices `s, 2s, 3s, …` — comparing only the *last* element of each block — until you find a block whose end is `≥ target`. The target, if present, must lie in that block, so scan its `s` elements linearly. The array must be sorted for "the block end brackets the target" to hold.

```text
target = 23, s = 3
[ 4 | 8 | 15 || 16 | 23 | 42 ]
       jump 1: a[2]=15 < 23 → jump
       jump 2: a[5]=42 ≥ 23 → target is in (a[2], a[5]]
       linear scan block [3..5]: 16, 23 ✓
```

## Implementation

```ts
function jumpSearch(a: number[], x: number): number {
  const n = a.length;
  if (n === 0) return -1;
  const step = Math.floor(Math.sqrt(n));   // optimal block size ≈ √n
  let prev = 0;                             // start of the current block
  let bound = step;                         // one past the block we probe

  // Phase 1: jump forward until the block END is >= x (or we run off the array).
  while (bound < n && a[Math.min(bound, n) - 1] < x) {
    prev = bound;
    bound += step;
  }
  // Phase 2: linear scan inside the found block [prev, min(bound, n)).
  const end = Math.min(bound, n);
  for (let i = prev; i < end; i++) {
    if (a[i] === x) return i;
  }
  return -1;
}
```

The two phases mirror the two costs we are balancing: `n/s` jumps in phase 1, up to `s` comparisons in phase 2.

## Dry Run

Search `x = 23` in `[4, 8, 15, 16, 23, 42]` (`n = 6`, `step = ⌊√6⌋ = 2`):

| phase | prev | bound | probe index | a[·] | action |
|---|---|---|---|---|---|
| jump | 0 | 2 | 1 | 8  | `8 < 23` → prev=2, bound=4 |
| jump | 2 | 4 | 3 | 16 | `16 < 23` → prev=4, bound=6 |
| jump | 4 | 6 | 5 | 42 | `42 ≥ 23` → stop |
| scan | 4 | end=6 | i=4 | 23 | **found → return 4** |

Two jumps then one scan comparison. Searching an absent `x = 20` would land in the same block `[4,6)` and scan `23, 42` without a match → `-1`.

## Complexity

Let `s` be the block size. Phase 1 does at most `⌈n/s⌉` probes; phase 2 does at most `s − 1` comparisons. Total ≈ `n/s + s`. Minimize over `s`: `d/ds (n/s + s) = −n/s² + 1 = 0` ⟹ `s = √n`, giving a minimum of `2√n − 1 = Θ(√n)`.

- **Best `O(1)`** — target in the first block's first probe path.
- **Average / worst `Θ(√n)`** — with the optimal `s = √n`.
- **Space `O(1)`** — iterative, a handful of indices.

`Θ(√n)` is strictly between linear's `Θ(n)` and binary's `Θ(log n)`: for `n = 10⁶`, that is ~1000 probes versus ~20 for binary. So on a plain in-memory array, binary search is simply better.

## Variants & Trade-offs

- **Block size choice.** `√n` is optimal for a symmetric cost model. If forward jumps and backward scans have *different* costs, the optimum shifts — a larger block if jumps are pricier, smaller if the in-block scan is.
- **Two-level jump search.** Recurse: instead of a linear scan inside the block, jump-search the block too, yielding `O(n^(1/3))`, and so on. Taken to the limit this converges toward binary search — which is why binary is the endpoint, not jump search.
- **vs [[binary-search]].** Binary wins on random-access memory in both comparison count and asymptotics.
- **vs [[linear-search]].** Jump search dominates linear on sorted data by a `√n` factor.

## When to Use — and When Not

Jump search is a niche tool justified by a **non-uniform cost model**, not by comparison count:

- **Sequential / block storage where backward seeks are expensive.** On tape, a spinning disk, or any medium where you read forward in blocks, binary search's repeated random jumps — each potentially a costly seek in *both* directions — hurt. Jump search moves forward monotonically through phase 1 and backs up exactly once (the in-block scan is a short forward run), so it plays nicely with sequential prefetching and one-directional media.
- **When jumping is cheap but the compare/access is the bottleneck** and you still want to beat linear without binary's jump pattern.

Do **not** use it as a default in-memory search: on RAM with `O(1)` random access, [[binary-search]] is faster and asymptotically superior. Jump search is mostly of academic and historical interest, plus the specific I/O-cost scenarios above.

## Interview Pitfalls & Gotchas

- **Unsorted input** — like all ordered searches, it silently returns wrong answers on unsorted data.
- **Off-by-one in the block bounds** — the probe compares `a[bound-1]` (the block *end*), and the scan runs `[prev, min(bound, n))`. Getting `min(bound, n)` wrong reads out of bounds on the final partial block when `n` is not a perfect multiple of `s`.
- **Forgetting the last partial block** — when `n` is not a perfect square, the final block is shorter than `s`; clamp with `Math.min(bound, n)`.
- **Claiming it beats binary search on RAM** — it does not; be precise that its edge is only under sequential/block-access cost models.
- **Scanning forward from 0 instead of from `prev`** — the whole point is to scan only the *last* block, not the whole prefix.

## See Also

- [[binary-search]]
- [[linear-search]]
- [[searching-algorithms]]
- [[exponential-search]]
- [[big-o-vs-reality]]
