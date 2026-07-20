# Sliding Window

The sliding window maintains a contiguous range `[left, right)` over a sequence and slides it forward, updating an incremental aggregate instead of recomputing it from scratch. It turns the naïve O(n·k) or O(n²) "check every subarray/substring" into a single amortized O(n) pass. It is the specialization of the [[two-pointer]] technique to *contiguous-subrange* problems.

## When the Technique Is Valid

Two conditions must both hold, or the window is unsound:

1. **The answer is a contiguous subarray/substring**, not an arbitrary subset. Windows only ever represent runs of adjacent elements.
2. **Feasibility is monotone in window size** — for variable windows, if a window is valid then every smaller sub-window sharing an endpoint behaves predictably (shrinking restores validity; growing preserves the property you're extending). Concretely: "longest window with sum ≤ S" works because growing can only *increase* the sum, so once you exceed S, shrinking from the left is the only fix. This monotonicity is what licenses each pointer to move *only forward*.

Where monotonicity fails — e.g. arrays with negative numbers for a "sum ≤ S" target, since extending can *decrease* the sum — the plain window is invalid and you fall back to [[prefix-sums]] plus search, or a [[monotonic-stack-and-queue|monotonic deque]].

## Fixed vs Variable Windows

**Fixed size k.** The window width is constant; slide by adding the entering element and removing the leaving one.

```ts
function maxSumK(a: number[], k: number): number {
  let sum = 0;
  for (let i = 0; i < k; i++) sum += a[i];      // first window
  let best = sum;
  for (let r = k; r < a.length; r++) {
    sum += a[r] - a[r - k];                       // add entering, drop leaving — O(1)
    best = Math.max(best, sum);
  }
  return best;
}
```

**Variable size.** `right` expands the window; when an invariant breaks, `left` advances to shrink it. The template:

```ts
function longestAtMostS(a: number[], S: number): number {
  let left = 0, sum = 0, best = 0;
  for (let right = 0; right < a.length; right++) {
    sum += a[right];                    // expand
    while (sum > S) sum -= a[left++];   // shrink until valid again
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

## Why It's Amortized O(n)

The inner `while` loop looks nested, but `left` only ever increases and is bounded by `right ≤ n`. Across the *entire* run, `left` advances at most `n` times total and `right` advances exactly `n` times — **each index enters the window once and leaves at most once**. So total work is O(n) amortized (aggregate method — see [[amortized-analysis]]), not O(n²), despite the nested-loop appearance. This "each element touched a constant number of times" argument is the recurring justification for the whole two-pointer family.

## Hash-Map / Counter-Assisted Windows

When the invariant is about *contents* rather than a numeric aggregate — "longest substring without repeats," "smallest window containing all chars of T," "at most K distinct" — carry a frequency map keyed by element. Expanding increments a count; shrinking decrements and deletes at zero. A running `distinct` counter or a `needed`/`formed` pair tracks satisfaction in O(1) per step.

- **At most K distinct** is directly a variable window (shrink while `distinct > K`).
- **Exactly K distinct** = `atMost(K) − atMost(K−1)` — a standard reduction, since "exactly" is not monotone but the difference of two monotone quantities is.

Map operations are O(1) average (see [[hash-maps-and-sets]]), keeping the whole scan O(n) average.

## Complexity

- **Time**: O(n) amortized for both fixed and variable windows; with a hash map, O(n) average (O(n·cost-of-hash) worst case under adversarial collisions).
- **Space**: O(1) for numeric windows; O(k) or O(alphabet) for counter-assisted windows.

## Relation to Neighboring Techniques

Sliding window *is* same-direction [[two-pointer]] with an aggregate maintained over the enclosed range. When the aggregate you need is a running max/min that isn't cheaply reversible on shrink, upgrade to a [[monotonic-stack-and-queue|monotonic deque]] — the "sliding window maximum" problem is exactly this, achieving O(n) by evicting dominated elements. For subarray-sum-equals-K with negatives, drop the window entirely and use prefix sums with a hash map.

## Senior Pitfalls

- **Applying it to non-contiguous problems** (subsets, subsequences) — the window can't represent them; you need DP or backtracking.
- **Negatives breaking monotonicity**: a window for "sum ≥ S" or "sum ≤ S" with negative values is unsound — the shrink condition no longer restores validity.
- **Shrinking with the wrong loop**: `if` vs `while`. For fixed windows an `if` suffices; for variable windows you must `while`-shrink until the invariant holds, or the window drifts invalid.
- **Off-by-one in width**: window length is `right − left + 1` for inclusive `right`, `right − left` for half-open — pick one convention and hold it.
- **Forgetting to delete zero-count keys** in a distinct-counter window, corrupting the `distinct` count.

## See Also

- [[two-pointer]]
- [[monotonic-stack-and-queue]]
- [[prefix-sums]]
- [[kadanes-algorithm]]
- [[hash-maps-and-sets]]
- [[amortized-analysis]]
