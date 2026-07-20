# Two-Pointer Technique

The two-pointer technique walks two indices over a sequence, using their relative movement to prune an O(n²) search down to a single O(n) pass with O(1) extra space. It solves the class of problems where a nested double loop is the obvious answer but the structure of the data (usually sortedness or monotonicity) lets you *never* revisit a discarded position.

## Two Shapes: Converging vs Same-Direction

**Opposite-ends / converging.** Start `lo = 0`, `hi = n-1`, move them toward each other based on a comparison. Because each step advances one pointer by one and they only ever move inward, the loop runs at most `n` times total.

```ts
function pairSum(a: number[], target: number): [number, number] | null {
  let lo = 0, hi = a.length - 1;          // requires a sorted ascending
  while (lo < hi) {
    const s = a[lo] + a[hi];
    if (s === target) return [lo, hi];
    if (s < target) lo++;                 // too small → need a bigger left value
    else hi--;                            // too big  → need a smaller right value
  }
  return null;
}
```

The correctness argument is the key insight: if `a[lo] + a[hi] < target`, then `a[lo]` paired with *anything ≤ a[hi]* is also too small, so `lo` can never be part of a solution with any smaller right index — discard it forever. That monotone-elimination invariant is what makes the single pass sound.

**Same-direction (slow/fast).** Both pointers move forward; a `write`/`slow` pointer lags a `read`/`fast` pointer, compacting or filtering in place. This is the in-place-mutation workhorse and is distinct from the cycle-detection [[fast-slow-pointers|fast/slow pointer]] variant on linked lists (same name, different mechanism).

## The Sorted-Array Prerequisite

Converging two-pointer relies on **monotonicity**: as one pointer moves, the quantity you compare against changes in a predictable direction. On an unsorted array that guarantee evaporates and the technique is simply wrong — you'd skip valid pairs. So the honest cost of "two-pointer pair sum" is O(n log n): the O(n) scan plus the sort that earns the monotonicity. If the array is already sorted (or you need a hash-based O(n) solution on unsorted data), account for that up front. Same-direction patterns (dedup, partition) do *not* require sorting.

## Canonical Patterns

- **Pair / two-sum on sorted input** — the converging template above; O(n).
- **In-place dedup of a sorted array** — slow pointer marks the end of the unique prefix; fast pointer scans; write only when `a[fast] !== a[slow]`. O(n), O(1).
- **Partition / Dutch National Flag** — three pointers (`low`, `mid`, `high`) sort an array of three key values (e.g. 0/1/2) in **one pass, O(n), O(1)**, swapping the `mid` element toward the correct end. This is the partition core reused by quickselect and quicksort — see [[sorting-algorithms]].
- **3-sum** — fix index `i`, then run a converging two-pointer over the remaining sorted suffix: O(n²) overall, beating the O(n³) triple loop, with careful duplicate skipping.
- **Merging two sorted arrays / interval overlap** — advance whichever pointer trails.

## Complexity

Both shapes are **O(n) time, O(1) extra space** for the scan itself. The pointers collectively traverse the array a constant number of times; no position is processed more than a bounded number of times. When a sort is a prerequisite, the total is dominated by O(n log n). This O(1)-space profile is the technique's headline advantage over a hash-set solution, which buys O(n) time on unsorted data at the cost of O(n) space.

## Two-Pointer vs Sliding Window vs Binary Search

Two-pointer, [[sliding-window]], and [[binary-search]] are a family: all three exploit ordering to avoid rescanning. Sliding window is the special case where both pointers move the *same* direction and you maintain an aggregate over the enclosed window (see that note for the amortized argument). Binary search is the degenerate case of a single interval `[lo, hi]` collapsing by halving rather than by unit steps. Reach for two-pointer when a pairing/partition over a monotone structure is the goal; sliding window when a *contiguous subrange aggregate* is; binary search when you're locating a threshold.

## Senior Pitfalls

- **Forgetting the sort** on converging patterns — the single most common correctness bug; the code runs and returns wrong answers.
- **Duplicate handling in 3-sum / k-sum**: after recording a hit you must advance *past* runs of equal values on both pointers, or you emit duplicate triples. This skip logic is where most implementations are subtly wrong.
- **Loop-boundary confusion**: `while (lo < hi)` vs `<=`. For pair-finding you want `<` (an element shouldn't pair with itself); for partition scans the condition differs. Getting this wrong causes off-by-one or an out-of-bounds read.
- **Assuming O(n) hides the sort** — quote O(n log n) when input isn't pre-sorted; interviewers probe exactly this.
- **Mutating while converging** without preserving the invariant that already-passed positions stay valid.

## See Also

- [[sliding-window]]
- [[binary-search]]
- [[fast-slow-pointers]]
- [[sorting-algorithms]]
- [[hash-maps-and-sets]]
