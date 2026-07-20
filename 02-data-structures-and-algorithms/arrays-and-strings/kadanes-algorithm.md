# Kadane's Algorithm

Kadane's algorithm finds the maximum-sum contiguous subarray in a single O(n) pass with O(1) space. It's the textbook example of turning an O(n²) (or naïve O(n³)) brute force into linear time via a one-dimensional dynamic-programming recurrence — the smallest, cleanest [[dynamic-programming|DP]] worth memorizing.

## The Idea: Best-Ending-Here

The trick is to reframe the global question ("best subarray anywhere") as a local one that composes. Let `best[i]` = the maximum sum of a subarray **ending exactly at index i**. Every subarray ends somewhere, so the global answer is `max over i of best[i]`. And `best[i]` has a one-line recurrence: the best subarray ending at `i` is either the element alone, or the element appended to the best subarray ending at `i-1`:

```
best[i] = max(a[i], best[i-1] + a[i])
```

The decision "extend or restart" is purely local: if the running sum `best[i-1]` has gone negative, it can only hurt, so drop it and restart at `a[i]`. Because `best[i]` depends only on `best[i-1]`, you don't need the array — collapse it to a scalar.

```ts
function maxSubarray(a: number[]): number {
  let bestEndingHere = a[0];      // best[i-1], rolled into a scalar
  let best = a[0];                // global answer
  for (let i = 1; i < a.length; i++) {
    bestEndingHere = Math.max(a[i], bestEndingHere + a[i]); // extend or restart
    best = Math.max(best, bestEndingHere);                  // track global max
  }
  return best;
}
```

## Complexity

O(n) time — one pass, O(1) work per element — and O(1) space (two scalars). This is optimal: any correct algorithm must inspect every element (a single unseen large positive changes the answer), so Ω(n) is a lower bound. It also handily beats the divide-and-conquer O(n log n) solution to the same problem — a rare case where DP is both simpler *and* asymptotically better than [[divide-and-conquer]].

## The All-Negative Edge Case

The most common Kadane bug is initializing `best = 0`. If every element is negative, the true maximum subarray is the single least-negative element, but `best = 0` wrongly reports `0` (an empty subarray). **Initialize `best` and `bestEndingHere` to `a[0]`** (or `-Infinity`), and iterate from index 1. The `Math.max(a[i], …)` form already enforces non-empty subarrays. Only if the problem *explicitly* allows the empty subarray should `0` be a valid floor — clarify this; it's a deliberate interview trap.

## Variants

- **With indices.** Track a candidate start that resets whenever you restart (`bestEndingHere` takes `a[i]` alone), and commit `[start, i]` to the answer whenever `best` improves.
- **Maximum-product subarray.** Products aren't monotone under multiplication — a large negative times another negative becomes a large positive. So carry **both** the running max *and* running min ending here, and swap them when `a[i] < 0`. `curMax = max(a[i], a[i]*prevMax, a[i]*prevMin)` (and symmetrically for `curMin`). Still O(n)/O(1); zeros force a restart of both.
- **Circular array.** The best wrap-around subarray equals `totalSum − (minimum subarray sum)` — invert the signs and run Kadane for the minimum. The answer is `max(standardKadane, total − minSubarray)`, with the caveat that if all elements are negative the `total − minSubarray` branch yields the empty subarray and must be discarded (return the standard Kadane result).
- **2-D maximum submatrix.** Fix a pair of top/bottom rows, collapse the columns between them into a 1-D array of column sums, and run Kadane across it — O(rows² · cols).

## Framing as Dynamic Programming

Kadane is DP with the two hallmark properties. **Optimal substructure**: the best subarray ending at `i` is built from the best ending at `i-1`. **Overlapping subproblems**: `best[i]` reuses `best[i-1]`. The scalar rolling of the DP table (keeping only the previous state) is the standard space-optimization every 1-D DP admits — see [[dynamic-programming]] and, for the general reuse pattern, [[07-performance-engineering/memoization|Memoization]]. Recognizing "best-ending-here" as a DP state is the transferable skill; the same shape reappears in longest-increasing-run, house-robber, and best-time-to-buy-sell-stock.

## Senior Pitfalls

- **`best = 0` initialization** — breaks all-negative inputs (the #1 mistake).
- **Confusing max-*sum* with max-*product*** — reusing the sum recurrence for products is wrong; you must track min too because of sign flips.
- **Contiguity assumption** — Kadane solves the *contiguous* subarray problem. Maximum-sum *subsequence* (non-contiguous) is trivially "sum of all positives" and is a different problem.
- **Circular edge case** — forgetting to special-case the all-negative wrap, which otherwise returns 0 from an empty subarray.
- **Returning the sum when indices are needed** — track the start on restart, not after the fact.

## See Also

- [[dynamic-programming]]
- [[prefix-sums]]
- [[sliding-window]]
- [[divide-and-conquer]]
- [[07-performance-engineering/memoization|Memoization]]
