# Exponential Search

Exponential search (also called doubling search or galloping search) finds a target in a **sorted** sequence by first *doubling* an index bound — `1, 2, 4, 8, …` — until it overshoots the target, then running a plain binary search inside the last bracket it found. Its signature strengths are **unbounded or unknown-size** inputs (infinite sorted streams, structures without a cheap length) and cases where the target sits near the front — it finds the range in `O(log i)`, where `i` is the target's index, rather than `O(log n)` over the whole array.

## The Idea

Two phases. **Phase 1 (find range):** probe indices `1, 2, 4, 8, 16, …`, doubling each time, until `a[bound] ≥ target` (or you run past the end). Now the target, if present, lies in `(bound/2, bound]`. **Phase 2 (binary search):** binary-search that bracket. Doubling reaches an index `≥ i` in `⌈log₂ i⌉` steps, and the bracket it isolates has width `≤ i`, so the binary search over it is also `O(log i)`.

```text
find 23 in [4, 8, 15, 16, 23, 42, …]
bound=1: a[1]=8  < 23 → double
bound=2: a[2]=15 < 23 → double
bound=4: a[4]=23 ≥ 23 → stop. range = (2, 4]  → binary-search [3..4]
```

## Implementation

```ts
function exponentialSearch(a: number[], x: number): number {
  const n = a.length;
  if (n === 0) return -1;
  if (a[0] === x) return 0;

  // Phase 1: double the bound until a[bound] >= x or we pass the end.
  let bound = 1;
  while (bound < n && a[bound] < x) bound *= 2;

  // Phase 2: binary search the bracket (bound/2, min(bound, n-1)].
  let lo = Math.floor(bound / 2);
  let hi = Math.min(bound, n - 1);
  while (lo <= hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (a[mid] === x) return mid;
    if (a[mid] < x) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}
```

For an **unbounded** source (an infinite stream or a length-unknown structure), drop `n`: replace `bound < n` with a probe that returns `+∞` past the end, so phase 1 discovers a finite upper bracket without ever needing the total size.

## Dry Run

Search `x = 23` in `[4, 8, 15, 16, 23, 42, 99]` (`n = 7`):

| phase | bound | probe | a[·] | action |
|---|---|---|---|---|
| double | 1 | a[1] | 8  | `8 < 23` → bound = 2 |
| double | 2 | a[2] | 15 | `15 < 23` → bound = 4 |
| double | 4 | a[4] | 23 | `23 ≥ 23` → stop |

Bracket: `lo = ⌊4/2⌋ = 2`, `hi = min(4, 6) = 4`. Binary search `[2, 4]`:

| step | lo | hi | mid | a[mid] | action |
|---|---|---|---|---|---|
| 1 | 2 | 4 | 3 | 16 | `16 < 23` → `lo = 4` |
| 2 | 4 | 4 | 4 | 23 | **found → return 4** |

Three doubling probes + two binary steps. Because the target is at index 4, the work scales with `log 4`, not `log 7` — and the gap widens dramatically when a huge array has its target near the front.

## Complexity

Let `i` be the (0-based) index of the target.

- **Phase 1** halts at the first power of two `≥ i`, so it takes `⌈log₂ i⌉ + 1 = O(log i)` probes.
- **Phase 2** searches a bracket of width `bound − bound/2 = bound/2 ≤ i`, so binary search costs `O(log i)`.
- **Total time `O(log i)`.** Best case `O(1)` (target at index 0 or 1). Worst case `O(log n)` (target at the end, `i ≈ n`) — never worse than binary search, and much better when `i ≪ n`.
- **Space `O(1)`** — iterative.

The key asymptotic contrast with plain [[binary-search]]: binary is `O(log n)` and *needs `n` up front* to set `hi`; exponential is `O(log i)` and *discovers a usable bound itself*, which is why it is the tool for unbounded inputs.

## Variants & Trade-offs

- **Galloping in merge/Timsort.** Timsort's merge step uses exactly this "gallop" — double the stride to find where one run's element falls in the other — to skip long already-ordered stretches in `O(log gap)`.
- **Backward exponential search.** Double *backward* from a hint position when the target is expected just below a known index (e.g. resuming a previous search).
- **vs [[jump-search]].** Both find a bracket then finish locally, but jump search uses *fixed* `√n` strides (`Θ(√n)`) while exponential uses *geometric* strides (`O(log i)`); exponential is asymptotically far better and does not need `n`.
- **Base of the doubling.** Growing by `×3` or `×k` changes constants only; `×2` is standard.

## When to Use — and When Not

- **Unbounded / unknown-size sorted data** — infinite or streaming sorted sequences, or structures where computing length is `O(n)` or impossible. This is the canonical use: exponential search brackets the answer without ever asking "how long is it?".
- **Target expected near the front** — `O(log i)` beats `O(log n)` when `i ≪ n`; searching the first few entries of a billion-element sorted log is nearly free.
- **As a range-finder feeding binary search** on any sorted structure with cheap forward access.

Do **not** bother when `n` is known, modest, and the target is uniformly located — plain [[binary-search]] is simpler and the same `O(log n)`. And like every ordered search, it demands sorted input.

## Interview Pitfalls & Gotchas

- **Clamping the upper bracket** — `bound` overshoots past the end; `hi` must be `min(bound, n - 1)` or the binary phase reads out of bounds.
- **Wrong lower bracket** — start the binary search at `bound/2`, not `0`; starting at `0` throws away the whole point (you would re-search the front you already cleared).
- **Integer overflow of `bound`** — repeated doubling can overflow a fixed-width int in other languages before it reaches `n`; in JS the float is safe to `2⁵³` but guard against runaway loops on truly infinite sources.
- **Handling `x` at index 0** — the `bound` starts at 1, so check `a[0]` first (done above) or the doubling loop skips it.
- **Unsorted input** — silently wrong, as with all binary-family searches.

## See Also

- [[binary-search]]
- [[jump-search]]
- [[searching-algorithms]]
- [[linear-search]]
- [[sorting-algorithms]]
