# Ternary Search

Ternary search splits the search range into **three** parts using two probes per step. It comes in two flavors that are easy to conflate: (a) the *important* one — finding the extremum (max or min) of a **unimodal function**, where no binary variant applies; and (b) a three-way split of a **sorted array** for exact lookup, which is a textbook curiosity because it does *more* comparisons than binary search, not fewer. Learn the function form; understand why the array form loses.

## The Idea

**On a unimodal function** `f` (strictly increasing then strictly decreasing, so it has one peak — or the mirror for a valley), you cannot binary-search: a single sample does not tell you which side the peak is on. But *two* interior probes do. Pick `m1` and `m2` with `lo < m1 < m2 < hi`. Comparing `f(m1)` and `f(m2)`:

- `f(m1) < f(m2)` → the peak is not left of `m1`; discard `[lo, m1]`, set `lo = m1`.
- `f(m1) > f(m2)` → the peak is not right of `m2`; discard `[m2, hi]`, set `hi = m2`.

Either way one of the three thirds is eliminated. Repeat until the interval is tiny.

```text
unimodal f, peak somewhere in [lo, hi]
lo -------- m1 -------- m2 -------- hi
         f(m1) < f(m2)?  → keep [m1, hi]   (peak is right of m1)
         f(m1) > f(m2)?  → keep [lo, m2]   (peak is left of m2)
```

## Implementation

```ts
// Maximize a unimodal function over [lo, hi] to within `eps`.
function ternaryMaxArg(
  f: (x: number) => number,
  lo: number,
  hi: number,
  eps = 1e-9,
): number {
  while (hi - lo > eps) {
    const m1 = lo + (hi - lo) / 3;   // first third
    const m2 = hi - (hi - lo) / 3;   // second third
    if (f(m1) < f(m2)) lo = m1;      // peak lies in [m1, hi]
    else hi = m2;                    // peak lies in [lo, m2]
  }
  return (lo + hi) / 2;              // midpoint of the collapsed interval
}
```

For a *discrete* array-index version, use integer `m1 = lo + (hi-lo)/3`, `m2 = hi - (hi-lo)/3` and loop while `lo < hi`; but for a plain sorted-array exact lookup you should reach for [[binary-search]] instead — see the comparison below.

## Dry Run (Unimodal Function)

Maximize `f(x) = −(x − 2)²  + 5` (a downward parabola peaking at `x = 2`, value 5) on `[0, 6]`:

| step | lo | hi | m1 = lo+(hi−lo)/3 | m2 = hi−(hi−lo)/3 | f(m1) | f(m2) | keep |
|---|---|---|---|---|---|---|---|
| 1 | 0.00 | 6.00 | 2.00 | 4.00 | 5.00 | 1.00 | `f(m1)>f(m2)` → `hi=4` |
| 2 | 0.00 | 4.00 | 1.33 | 2.67 | 4.56 | 4.56 | `f(m1)≥f(m2)` → `hi=2.67` |
| 3 | 0.00 | 2.67 | 0.89 | 1.78 | 3.76 | 4.95 | `f(m1)<f(m2)` → `lo=0.89` |
| … | | | | | | | interval collapses toward `x=2` |

The interval shrinks by a factor of `2/3` each step, homing in on the true peak `x = 2`. No ordering of the *outputs* exists to binary-search — only the unimodal shape makes the two-probe elimination valid.

## Why Three-Way Array Lookup Loses to Binary

For a *sorted array* exact search you *could* split into thirds: probe `m1` and `m2`, and either match, or recurse into one of three sub-arrays of size ~`n/3`. The recurrence is `T(n) = T(n/3) + O(1)`, giving `Θ(log₃ n)` **iterations** — fewer than binary's `log₂ n`. That sounds better, but count **comparisons**, which is the real cost:

- **Binary:** ~1 comparison per level × `log₂ n` levels = `log₂ n` comparisons.
- **Ternary:** up to 2 comparisons per level × `log₃ n` levels = `2·log₃ n` comparisons.

Since `log₃ n = log₂ n / log₂ 3 ≈ 0.631·log₂ n`, ternary does `2 × 0.631 × log₂ n ≈ 1.26·log₂ n` comparisons — about **26% more** than binary. Fewer levels, but each level is twice as expensive, and the arithmetic does not net out in ternary's favor. Splitting into `k` ways is `((k−1)/log₂ k)·log₂ n` comparisons, minimized at `k = 2`. This is the general lesson: **binary is the optimal fan-out for comparison-based lookup.**

## Complexity

- **Unimodal optimization:** each iteration removes one-third, so the interval shrinks by `2/3`; reaching precision `eps` over a range `R` takes `⌈log₁.₅(R/eps)⌉ = O(log(R/eps))` iterations, **2 function evaluations each**. Space `O(1)`.
- **Array lookup:** `Θ(log₃ n)` probes but `≈1.26·log₂ n` comparisons — asymptotically `Θ(log n)`, with a **worse constant** than binary. Space `O(1)` iterative.

## Variants & Trade-offs

- **Golden-section search.** The smart replacement for ternary optimization: it reuses one probe across iterations by placing samples at the golden ratio, needing only **1** new `f` evaluation per step instead of 2 — nearly halving evaluations when `f` is expensive. Prefer it over ternary for real optimization work.
- **Ternary on integers / discrete unimodal arrays.** Valid when the array is strictly unimodal; watch the termination when the window shrinks to 2–3 elements (finish with a linear check).
- **vs [[binary-search]].** Use binary for sorted-array lookup, always. Use ternary (or golden-section) only when the target is an *extremum of a unimodal function*, where binary search simply does not apply.

## When to Use — and When Not

Reach for ternary search to **optimize a unimodal function** with no closed form: peak of a projectile's range vs. launch angle, the distance-minimizing point along a path, or a "sweet spot" parameter that first improves then worsens a metric. Do **not** use it for ordered-array membership (binary is strictly better) and do **not** apply it to non-unimodal functions — multiple local extrema break the elimination rule and it converges to the wrong point.

## Interview Pitfalls & Gotchas

- **Applying it to non-unimodal `f`.** The two-probe elimination is only valid for a single peak/valley; multiple maxima silently mislead it.
- **Claiming it beats binary for array lookup.** It does not — it does ~26% more comparisons; be ready to derive `(k−1)/log₂ k` and show `k=2` is optimal.
- **Wrong extremum direction.** The code above maximizes; for a minimum, flip the comparison (`f(m1) > f(m2)` keeps the left).
- **Termination on floats.** Loop on `hi - lo > eps`, not on exact equality — floating point never converges exactly; also cap iterations to avoid a stall.
- **Off-by-one in the discrete version** when the window is 1–2 wide — probes can coincide; finish with a small linear scan.

## See Also

- [[binary-search]]
- [[searching-algorithms]]
- [[linear-search]]
- [[divide-and-conquer]]
- [[big-o-vs-reality]]
