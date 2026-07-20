# Binary Search

Binary search locates a target — or a boundary position — in a **sorted** sequence by repeatedly halving the search interval, turning `O(n)` into `O(log n)`. Its deeper value in real work is the *pattern*: any problem where a monotone predicate flips from false to true once can be solved by binary-searching the answer, even when there is no literal array in sight.

## The Idea & Invariant

Maintain a candidate interval and shrink it by half each step. The discipline that kills every binary-search bug is picking **one** interval convention and never violating it. I use **half-open `[lo, hi)`**: `hi` starts at `n` (one past the end, *exclusive*), and the invariant is "the answer, if present, lies in `[lo, hi)`." Loop while `lo < hi`. On each step, compare the midpoint and discard the half that cannot contain the answer.

```text
find 23 in [4, 8, 15, 16, 23, 42]
[lo........hi)   mid=3 → a[3]=16 < 23, discard left
       [lo..hi)  mid=5 → a[5]=42 > 23, discard right
       [lo hi)   mid=4 → a[4]=23  ✓
```

## Implementation

The boundary-search forms are strictly more general than exact match and handle duplicates cleanly, so learn them and derive the rest:

```ts
// First index i with a[i] >= x  (insertion point). Invariant:
//   everything left of lo is  < x;  everything at/right of hi is >= x.
function lowerBound(a: number[], x: number): number {
  let lo = 0, hi = a.length;            // half-open [lo, hi), hi exclusive
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);  // overflow-safe floor midpoint
    if (a[mid] < x) lo = mid + 1;       // mid ruled out on the low side
    else hi = mid;                      // mid still a candidate
  }
  return lo;                            // == hi on exit
}

// First index i with a[i] > x.
function upperBound(a: number[], x: number): number {
  let lo = 0, hi = a.length;
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (a[mid] <= x) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

const contains = (a: number[], x: number) => {
  const i = lowerBound(a, x);
  return i < a.length && a[i] === x;
};
const count = (a: number[], x: number) => upperBound(a, x) - lowerBound(a, x);
```

**Progress guarantee (no infinite loop):** `lo = mid + 1` strictly advances `lo`, and `hi = mid` strictly shrinks `hi` because `mid < hi` always (the floor of `(hi-lo)/2` added to `lo` is `< hi` whenever `lo < hi`). The interval therefore narrows every iteration. Writing `lo = mid` instead of `mid + 1` is the single most common cause of a hang.

## The `mid` Overflow Nuance

The textbook bug is `mid = (lo + hi) / 2`: `lo + hi` can overflow a fixed-width integer — the defect that lived in the JDK's binary search for nine years. The fix `mid = lo + (hi - lo) / 2` never overflows because `hi - lo` fits. **TypeScript nuance:** JS `number` is a 64-bit float, so `lo + hi` only misbehaves past `2⁵³` — you will never hit it on array indices, but the correct idiom still matters the moment you binary-search over a large *numeric answer space* rather than indices. Use `lo + ((hi - lo) >> 1)` for an integer floor (the `>> 1` also coerces to int32, so keep values within that range on answer-space searches, or use `Math.floor(... / 2)`).

## Dry Run

`lowerBound([4, 8, 15, 16, 23, 42], 23)` on `[lo, hi) = [0, 6)`:

| step | lo | hi | mid | a[mid] | test `a[mid] < 23` | action |
|---|---|---|---|---|---|---|
| 1 | 0 | 6 | 3 | 16 | true  | `lo = 4` |
| 2 | 4 | 6 | 5 | 42 | false | `hi = 5` |
| 3 | 4 | 5 | 4 | 23 | false | `hi = 4` |
| — | 4 | 4 | — | — | — | `lo == hi` → return 4 |

Three iterations narrow `[0,6)` to the single boundary index 4; `a[4] === 23`, so `contains` is true. Searching an absent `x = 20` would exit with `lo = 4` as well (`a[3]=16 < 20 <= 23=a[4]`), and `contains` returns false — the same code cleanly yields the insertion point.

## Binary Search on the Answer (Parametric Search)

The real superpower: when you cannot index a sorted array but *can* define a **monotone predicate** `p(x)` — false for all `x` below some threshold `t`, true for all `x ≥ t` — binary-search the value `t` directly. The "array" is the conceptual sorted domain of candidate answers.

```ts
// Smallest integer capacity in [lo, hi] for which feasible() holds.
function minFeasible(lo: number, hi: number, feasible: (c: number) => boolean): number {
  while (lo < hi) {
    const mid = lo + Math.floor((hi - lo) / 2);
    if (feasible(mid)) hi = mid;   // mid works → search for something smaller
    else lo = mid + 1;             // mid fails → must go larger
  }
  return lo;
}
```

This solves the "minimum/maximum X such that…" family. Cost is `O(log(range) · cost(predicate))`; the actual work is *proving* the predicate is monotone.

## Complexity

`O(log n)` comparisons, `O(1)` extra space (iterative; a recursive version costs `O(log n)` stack). The recurrence `T(n) = T(n/2) + O(1)` solves to `Θ(log n)` — each comparison eliminates half the remaining candidates, so `log₂ n` halvings exhaust the range. Best, average, and worst are all `Θ(log n)` (a lucky early hit only helps exact-match search, not boundary search). Prerequisite: the data must be **sorted** — `O(n log n)` if you sort first (see [[sorting-algorithms]]), amortized only across many searches sharing one sort.

## Interview Pitfalls & Gotchas

- **Unsorted input** — silently returns garbage; the algorithm assumes sortedness and cannot detect its violation.
- **Boundary mixing** — combining `hi = n` with `lo <= hi` is the classic off-by-one / out-of-bounds; commit to `[lo, hi)` *or* `[lo, hi]`.
- **Infinite loop** from `lo = mid` (must be `mid + 1`) or a `mid` that fails to advance.
- **Duplicates** — exact-match search returns *an* index, not the first or last; use `lowerBound`/`upperBound` when position matters.
- **Cache reality** — for small `n`, non-local jumps and branch mispredicts can lose to a linear scan despite the optimal `O(log n)` (see [[big-o-vs-reality]]).

## See Also

- [[sorting-algorithms]]
- [[two-pointer]]
- [[binary-search-trees]]
- [[exponential-search]]
- [[searching-algorithms]]
- [[big-o-vs-reality]]
