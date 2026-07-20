# Amortized Analysis

Amortized analysis bounds the cost of a *sequence* of operations, then divides by the number of operations to get a per-operation guarantee. It solves the problem that worst-case-per-operation is often pessimistic and misleading: some operations are occasionally expensive but *pay forward* work that makes many subsequent operations cheap. The amortized bound is a rigorous guarantee — it holds for *every* sequence, with no probability involved.

## Three Methods

**Aggregate method.** Bound the total cost of any sequence of `m` operations by `T(m)`, then the amortized cost per operation is `T(m)/m`. Simple, but gives one number for all operation types.

**Accounting (banker's) method.** Assign each operation an *amortized charge* that may differ from its real cost. When the charge exceeds real cost, deposit the surplus as *credit* stored on data-structure elements; when real cost exceeds the charge, pay the difference from stored credit. The invariant you must maintain: **credit is never negative.** If it holds, the total amortized charge upper-bounds the total real cost.

**Potential method.** Define a potential function `Φ` mapping each state to a nonnegative number (the "stored energy"), with `Φ(initial) = 0` and `Φ ≥ 0` always. The amortized cost of an operation is `ĉ = c_real + Φ(after) − Φ(before)`. Summing telescopes: `Σĉ = Σc_real + Φ(final) − Φ(initial) ≥ Σc_real`. Choose `Φ` so expensive operations release exactly the potential they need.

## Canonical Example: Dynamic-Array Doubling

A [[dynamic-arrays|dynamic array]] appends in `O(1)` until full, then reallocates a larger buffer and copies everything — an `O(n)` step. The subtlety: how large is the new buffer? **Doubling** (geometric growth) gives `O(1)` amortized push; a fixed increment (e.g. +10) does *not* — it gives `O(n)` amortized.

**Aggregate proof for doubling.** Start empty, do `n` pushes. Cheap pushes cost `n` total. Resizes happen at sizes `1, 2, 4, ..., ≤ n`, copying `1 + 2 + 4 + ... + 2^k` elements. That geometric sum is `< 2n`. Total work `< n + 2n = 3n = O(n)`, so amortized cost per push is `O(1)`.

**Accounting proof.** Charge each push **3** units. One unit pays the immediate write. Two units are banked on the element just written. When the array (size `k`) fills and doubles, the `k/2` elements added since the last resize each carry 2 banked units — exactly enough to pay for copying all `k` elements. Credit stays nonnegative, so amortized cost is a constant 3.

```ts
class Vec<T> {
  private buf: T[] = new Array(1);
  private len = 0;
  push(x: T) {
    if (this.len === this.buf.length) {      // rare: O(n) copy
      const bigger = new Array(this.buf.length * 2); // geometric growth
      for (let i = 0; i < this.len; i++) bigger[i] = this.buf[i];
      this.buf = bigger;
    }
    this.buf[this.len++] = x;                 // common: O(1)
  }
}
```

Why the growth factor must be geometric: with fixed increment `d`, resizes occur every `d` pushes and copy an arithmetic series `∝ n²/d`, so `n` pushes cost `Θ(n²)` — `Θ(n)` amortized per push. Any factor `> 1` gives `O(1)`; the factor trades wasted memory against copy frequency (many implementations use 1.5× to reduce fragmentation and allow buffer reuse).

## Amortized ≠ Average-Case

This is the most important distinction and the most common conflation. **Average-case** is a probabilistic statement: expected cost over a *distribution of inputs*, and an adversary picking bad inputs can defeat it. **Amortized** is a deterministic worst-case guarantee over a *sequence*: no distribution, no expectation, no adversarial escape. Every possible sequence of `n` dynamic-array pushes costs `O(n)`, period. Hash-table `O(1)` lookup is average-case (a bad hash or adversarial keys break it — see [[hash-tables]] and [[collision-resolution]]); dynamic-array `O(1)` push is amortized (unbreakable).

## Where Amortized Bounds Hurt

The guarantee is about *total* work, not *individual* latency. In latency-sensitive or real-time systems the occasional `O(n)` spike is exactly what kills you:

- A [[hash-tables|hash table]] with amortized-`O(1)` insert will, on one unlucky insert, rehash millions of keys — a multi-millisecond stall in the middle of a request.
- Dynamic-array growth can double a gigabyte buffer, triggering a large allocation and copy (and GC pressure — see [[07-performance-engineering/garbage-collection|Garbage Collection]]) at an unpredictable moment.
- Trading floors, audio DSP, and game frame loops care about the *p99.9 tail*, not the mean. Here you prefer structures with good *worst-case-per-op* bounds, or you pre-size, or you use incremental/de-amortized variants that spread the resize work across many operations to cap the spike. See [[07-performance-engineering/latency-vs-throughput|Latency vs Throughput]].

## See Also

- [[dynamic-arrays]]
- [[hash-tables]]
- [[asymptotic-notation]]
- [[collision-resolution]]
- [[07-performance-engineering/latency-vs-throughput|Latency vs Throughput]]
- [[07-performance-engineering/garbage-collection|Garbage Collection]]
