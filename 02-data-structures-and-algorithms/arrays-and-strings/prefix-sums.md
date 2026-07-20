# Prefix Sums

A prefix-sum (cumulative-sum) array precomputes running totals so that the sum of any contiguous range can be answered in O(1). It trades an O(n) preprocessing pass and O(n) extra space for turning a stream of range-sum queries from O(n) each into O(1) each — the canonical precompute-once-query-many pattern.

## The Idea and Invariant

Define `P[0] = 0` and `P[i] = a[0] + a[1] + … + a[i-1]`. The invariant is that `P[i]` holds the sum of the first `i` elements, so the **half-open** range `[l, r)` is:

```ts
function buildPrefix(a: number[]): number[] {
  const P = new Array(a.length + 1).fill(0);
  for (let i = 0; i < a.length; i++) P[i + 1] = P[i] + a[i]; // O(n)
  return P;
}
// sum of a[l..r-1] in O(1):
const rangeSum = (P: number[], l: number, r: number) => P[r] - P[l];
```

The `n+1`-length, 1-based convention (`P[0] = 0`) is worth adopting deliberately: it removes the `l === 0` special case that plagues 0-based implementations. Every range query is then one subtraction — the telescoping cancellation `P[r] - P[l]` leaves exactly `a[l] + … + a[r-1]`.

## Complexity

- **Build**: O(n) time, O(n) space.
- **Range-sum query**: O(1).
- **Point update** (`a[i] += δ`): O(n) — every prefix from `i+1` onward must change. This is the fatal weakness for mixed read/write workloads and the reason segment trees and Fenwick trees exist (below).

Break-even vs recompute: naïve range sum is O(n) per query, so for `q` queries prefix sums cost O(n + q) instead of O(nq). Even a single query over a stable array benefits little, but for `q ≫ 1` on a static array the win is decisive.

## Difference Arrays — the Dual for Range Updates

Prefix sums make *range queries / point updates* cheap. Its dual, the **difference array**, makes *range updates / point queries* cheap. To add `δ` to every element in `[l, r)`, record `d[l] += δ` and `d[r] -= δ`; after all updates, the prefix sum of `d` reconstructs the final array in O(n).

```ts
function applyRangeAdds(n: number, updates: [number, number, number][]): number[] {
  const d = new Array(n + 1).fill(0);
  for (const [l, r, delta] of updates) { d[l] += delta; d[r] -= delta; } // O(1) each
  const out = new Array(n).fill(0);
  for (let i = 0; i < n; i++) out[i] = (i ? out[i - 1] : 0) + d[i]; // finalize O(n)
  return out;
}
```

So `k` range updates cost O(k + n) total rather than O(kn) — the workhorse behind "add to a range" batch problems, interval booking sweeps, and 1-D imaging kernels.

## 2D Prefix Sums and Inclusion–Exclusion

Extend to a matrix with `P[i][j] = sum of the submatrix from (0,0) to (i-1,j-1)`. The rectangle sum with top-left `(r1,c1)` and bottom-right exclusive `(r2,c2)` uses **inclusion–exclusion**:

```
sum = P[r2][c2] − P[r1][c2] − P[r2][c1] + P[r1][c1]
```

You subtract the top and left overhanging strips, then add back the top-left corner that was subtracted twice. Build is O(mn) via `P[i][j] = a + P[i-1][j] + P[i][j-1] − P[i-1][j-1]`; each query is O(1). This generalizes to *d* dimensions with `2^d` terms per query (alternating signs), the reason it stays practical only for small `d`.

## When It Beats Recompute — and When It Doesn't

Use prefix sums when the array is **static** (or rarely mutated) and you face many range aggregates over a group operation with an inverse — sum, XOR, count. For min/max there is no inverse, so subtraction fails; use a sparse table (idempotent, O(1) after O(n log n)) or a segment tree instead. If the array is **frequently updated between queries**, prefix sums' O(n) update kills you.

## Relation to Segment Trees and Fenwick Trees

This is the static-vs-updatable axis. Prefix sums give O(1) query but O(n) update. A [[segment-trees-and-fenwick|Fenwick / segment tree]] gives O(log n) query *and* O(log n) update — strictly better when updates are frequent, strictly worse (log factor, more memory, worse locality) when they are not. Choose by workload: read-mostly → prefix sums; read/write mix → Fenwick/segment tree.

## Senior Pitfalls

- **Off-by-one on the range convention.** Mixing inclusive `[l, r]` with the half-open `P[r] − P[l]` formula is the most common bug; pick half-open and the size-`n+1` array and never special-case `l = 0`.
- **Integer overflow.** Cumulative sums grow fast; `n · max(a)` can exceed 32/64-bit range. In TS, `number` loses integer precision past 2^53 — use `BigInt` for large accumulations.
- **Applying it to non-invertible operations** (min/max) — subtraction only works for a group.
- **Rebuilding after every update** instead of switching to a Fenwick tree — a silent O(n) per mutation.

## See Also

- [[segment-trees-and-fenwick]]
- [[sliding-window]]
- [[kadanes-algorithm]]
- [[two-pointer]]
- [[07-performance-engineering/algorithmic-optimization|Algorithmic Optimization]]
