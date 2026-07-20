# Radix Sort

Radix Sort sorts fixed-width keys (integers, fixed-length strings) **digit by digit** — no key-to-key comparisons. Each pass sorts on one digit using a **stable** [[counting-sort]], and because the passes are stable, sorting from the least-significant digit up leaves the whole array sorted. It runs in `Θ(d·(n + b))` for `d` digits in base `b`, which beats `n log n` when keys are short and numerous.

## The idea / invariant

Sort on the least-significant digit (LSD) first, then the next, up to the most-significant. The **invariant after pass `p`**: the array is correctly sorted by the last `p` digits. A stable sort on digit `p+1` refines that ordering — elements agreeing on digit `p+1` keep their relative order, which is precisely their order by the lower `p` digits. This is why **per-digit stability is mandatory**: an unstable digit sort would scramble the work of every earlier pass, and the result would be garbage.

```
LSD pass on ones digit, then tens digit:
[170 45 75 90 802 24 2 66]
 ones -> [170 90 802 2 24 45 75 66]
 tens -> [802 2 24 45 66 170 75 90]  (802,2 have tens=0)
 hundreds -> [2 24 45 66 75 90 170 802]  sorted
```

## Implementation (LSD, base `b`, non-negative ints)

```ts
function radixSort(a: number[], base = 10): number[] {
  if (a.length === 0) return a;
  const max = Math.max(...a);
  // one pass per digit of the largest value
  for (let exp = 1; Math.floor(max / exp) > 0; exp *= base) {
    a = countingSortByDigit(a, exp, base);
  }
  return a;
}

// STABLE counting sort keyed on the digit at position `exp`.
function countingSortByDigit(a: number[], exp: number, base: number): number[] {
  const count = new Array(base).fill(0);
  for (const v of a) count[Math.floor(v / exp) % base]++;
  for (let d = 1; d < base; d++) count[d] += count[d - 1];  // prefix sums

  const out = new Array(a.length);
  for (let i = a.length - 1; i >= 0; i--) {   // RIGHT-TO-LEFT -> stable
    const digit = Math.floor(a[i] / exp) % base;
    out[--count[digit]] = a[i];
  }
  return out;
}
```

The inner routine is exactly counting sort over `b` buckets, and the right-to-left placement gives the stability the outer loop depends on.

## Dry run — two digit passes

Sort `[329, 457, 657, 839, 436, 720, 355]`, base 10.

**Pass 1 (ones digit):** keys `9,7,7,9,6,0,5` → grouped stably:
```
720, 355, 436, 457, 657, 329, 839
(0)  (5)  (6)  (7)  (7)  (9)  (9)   <- ones digit; 457 before 657 preserved
```

**Pass 2 (tens digit):** on the array above, tens digits are `2,5,3,5,5,2,3`:
```
720, 329, 436, 839, 355, 457, 657
(2)  (2)  (3)  (3)  (5)  (5)  (5)
```
Note `720` before `329` (both tens=2) — order from pass 1, where 720 < 329 on the ones digit, is preserved by stability.

**Pass 3 (hundreds):** digits `7,3,4,8,3,4,6` → `329, 355, 436, 457, 657, 720, 839` — fully sorted.

## Complexity

**Time — `Θ(d·(n + b))`.** There are `d` digit passes; each pass is a counting sort costing `Θ(n + b)`. With `d` and `b` treated as constants (fixed-width keys, fixed base) this is `Θ(n)` — linear, and independent of input order, so best = average = worst. For 32-bit integers sorted in base 256, `d = 4` and `b = 256`: four cheap passes regardless of `n`.

**Space — `Θ(n + b)`.** Each pass needs the `Θ(n)` output buffer and a `Θ(b)` count array. **Not in-place.**

**Stable — YES**, provided every digit pass is stable (it must be — see above).

**The `n log n` comparison, honestly.** Radix's `d` hides a `log` in disguise: to have `n` *distinct* keys you need `d ≥ log_b(n)`, so `d·(n+b)` is `Ω(n log n)` in the worst pathological case. Radix wins when `d` is a small fixed constant set by the key **width**, not by `n` — e.g. sorting a million 32-bit ints in 4 passes while a comparison sort needs `~20n` comparisons.

## LSD vs MSD

- **LSD** (above): process least-significant digit first, one full stable pass per digit, no recursion. Simple, cache-friendlier, great for fixed-width numeric keys. Every element is touched every pass.
- **MSD**: process most-significant digit first, then **recurse into each bucket** on the next digit. Naturally handles **variable-length strings** and can short-circuit once buckets have one element, but it's recursive and has more overhead. Used for string sorting (dictionary order).

## When to use / when not

- **Beats comparison sorts** for large `n` of fixed-width keys: 32/64-bit integers, IP addresses, fixed-length strings, dates as `YYYYMMDD`. Linear time, predictable, no worst case.
- **Loses** when keys are **few but huge** (`d` large, `n` small — the `d` factor dominates), when key width is unbounded/variable in a way that inflates `d`, or when the extra `Θ(n + b)` memory is unaffordable. General-purpose data with arbitrary comparable keys → use [[quick-sort]]/[[hybrid-sorts]].

## Variants & trade-offs

- **Base choice trades passes for buckets**: larger `b` → fewer digits `d` but a bigger `Θ(b)` count array. Base 256 (byte-at-a-time) is a common sweet spot for integers.
- Closely related to [[bucket-sort]] (both scatter by key structure) and built directly on [[counting-sort]].
- For strings, MSD radix underpins fast lexicographic sorts — see [[string-algorithms]].

## Interview pitfalls & gotchas

- **Using an unstable per-digit sort** — the number-one conceptual error; it destroys the ordering from previous passes.
- **Sorting MSD-first with LSD logic** (single non-recursive pass) — MSD requires recursing per bucket, or the low digits never get resolved.
- **Negative numbers** — plain radix assumes non-negative keys; handle signs by offsetting, or sort magnitudes and reverse the negatives.
- **Floats** — sort their IEEE-754 bit patterns with a sign-flip transform, not the raw bytes.
- **Claiming unconditional `O(n)`** — it's `Θ(d·(n+b))`; when `d` grows with `n` (distinct keys) the `log` reappears.

## See Also
- [[counting-sort]]
- [[bucket-sort]]
- [[string-algorithms]]
- [[sorting-algorithms]]
- [[quick-sort]]
