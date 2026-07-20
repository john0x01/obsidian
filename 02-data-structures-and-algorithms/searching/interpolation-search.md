# Interpolation Search

Interpolation search finds a target in a **sorted, numerically-keyed** array by *guessing where it should be* rather than always splitting in the middle. It models the array as a straight line from the low value to the high value and interpolates the probe position from the target's value — the way you open a phone book near "S" for "Smith" instead of dead center. On near-uniformly distributed keys this reaches expected `O(log log n)`; on skewed data it degrades all the way to `O(n)`.

## The Idea

Binary search always probes the midpoint — it is *data-agnostic*, ignoring how large the target is relative to the range. Interpolation search reads the values at the two ends and asks: if the keys rise linearly across `[lo, hi]`, what index should hold `x`? That estimate is a proportional position:

```text
pos = lo + (x − a[lo]) * (hi − lo) / (a[hi] − a[lo])
```

If `x` is near `a[lo]`, `pos` is near `lo`; if near `a[hi]`, near `hi`. On perfectly uniform data the first probe is often *the answer*.

```text
find 23 in [0, 10, 20, 23, 40, 90], lo=0 hi=5
pos = 0 + (23−0)*(5−0)/(90−0) = 115/90 ≈ 1 → probe a[1]=10 < 23, move lo up
```

## Implementation

```ts
function interpolationSearch(a: number[], x: number): number {
  let lo = 0, hi = a.length - 1;
  while (lo <= hi && x >= a[lo] && x <= a[hi]) {   // x must be within range
    if (a[lo] === a[hi]) {                          // flat segment: avoid /0
      return a[lo] === x ? lo : -1;
    }
    // Proportional probe; floor keeps it an integer index.
    const pos = lo + Math.floor(
      ((x - a[lo]) * (hi - lo)) / (a[hi] - a[lo])
    );
    if (a[pos] === x) return pos;
    if (a[pos] < x) lo = pos + 1;   // discard everything <= pos
    else hi = pos - 1;              // discard everything >= pos
  }
  return -1;
}
```

The `a[lo] === a[hi]` guard prevents a divide-by-zero when the remaining segment is constant, and the `x >= a[lo] && x <= a[hi]` loop condition both bounds the probe and short-circuits out-of-range targets immediately.

## Dry Run

Search `x = 23` in the near-uniform `[0, 10, 20, 30, 40, 50]` (`lo=0, hi=5`):

| step | lo | hi | pos formula | pos | a[pos] | action |
|---|---|---|---|---|---|---|
| 1 | 0 | 5 | `0 + ⌊23·5/50⌋ = ⌊2.3⌋` | 2 | 20 | `20 < 23` → `lo = 3` |
| 2 | 3 | 5 | `3 + ⌊(23−30)·2/20⌋` = `3 + ⌊−0.7⌋` | 2 | — | `pos < lo`… |

Here `23` is not in this array; step 2's interpolation drives `pos` below `lo`, the loop condition `x <= a[hi]` and `lo <= hi` catch it, and it returns `-1`. Now trace a hit — search `x = 30`: step 1 gives `pos = ⌊30·5/50⌋ = 3`, `a[3] = 30` → **found in one probe**. That single-probe hit on uniform data is exactly the behavior that yields `O(log log n)`.

## Complexity

- **Expected `O(log log n)`** on keys drawn from a (roughly) **uniform** distribution. The analysis: each probe shrinks the *expected* remaining range not by a constant factor but roughly to its square root, because a good linear estimate localizes the target within ~`√m` of the true position in a range of size `m`. The recurrence `T(m) = T(√m) + O(1)` unrolls to `O(log log n)`.
- **Worst case `O(n)`** — on skewed/exponential data (e.g. `[1, 2, 3, …, 1000, 10⁹]`), the linear model badly mis-estimates, the probe advances by roughly one element at a time, and it degenerates to a linear scan.
- **Space `O(1)`** — iterative.

Contrast with [[binary-search]], whose `Θ(log n)` is *guaranteed* and *distribution-independent*: it never asks what the values are, only their order, so no adversarial distribution can hurt it. Interpolation search trades that guarantee for a better expected case *when* the data cooperates.

## Variants & Trade-offs

- **Interpolation–binary hybrid.** Interpolate, but cap how far a single probe may move; if progress stalls, fall back to a binary step. This keeps the `O(log log n)` best case while restoring an `O(log n)` worst-case guard — the practical way to ship it.
- **Higher-order interpolation.** Fit a quadratic or spline instead of a line for non-linear-but-smooth key distributions; rarely worth the constant-factor and complexity cost.
- **vs [[binary-search]].** Binary needs only *order* (works on any comparable type); interpolation needs *arithmetic on the keys* (numeric, and ideally uniform). Use interpolation only when you know the distribution is friendly.

## When to Use — and When Not

Use interpolation search on **large, sorted, numeric arrays whose keys are approximately uniformly distributed** — dense integer IDs, timestamps sampled at a steady rate, uniformly random floats. There the `log log n` factor is a real win at scale (for `n = 10⁹`, `log log n ≈ 5` versus `log n ≈ 30`).

Avoid it when keys are non-numeric (no arithmetic to interpolate), when the distribution is skewed or unknown (worst-case `O(n)` and adversary-exploitable), or when `n` is small (the extra multiply/divide per probe is not worth it — [[binary-search]] or even [[linear-search]] wins on constants).

## Interview Pitfalls & Gotchas

- **Divide-by-zero** when `a[lo] === a[hi]` (a constant segment, common with duplicates) — guard it explicitly.
- **Overflow in the product** `(x - a[lo]) * (hi - lo)` — in fixed-width integer languages this multiplication overflows; in JS the 64-bit float is safe up to `2⁵³` but loses precision beyond it.
- **Assuming `O(log log n)` unconditionally** — that bound holds *only* on uniform data; on skewed input it is `O(n)`, a favorite interviewer follow-up.
- **`pos` out of `[lo, hi]`** — floating error or an out-of-range `x` can push `pos` outside the window; the loop guard `x >= a[lo] && x <= a[hi]` plus `lo <= hi` must contain it, or you index out of bounds.
- **Using it on non-uniform real data by default** — measure the distribution first; a hybrid with a binary fallback is the safe production choice.

## See Also

- [[binary-search]]
- [[searching-algorithms]]
- [[linear-search]]
- [[jump-search]]
- [[big-o-vs-reality]]
