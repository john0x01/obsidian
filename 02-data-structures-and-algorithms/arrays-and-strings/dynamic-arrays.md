# Dynamic Arrays

A dynamic array (C++ `vector`, Java `ArrayList`, Python `list`, JS `Array`) is a resizable sequence built on top of a fixed-size contiguous buffer. It solves the tension between the two things a fixed array can't give you at once: **O(1) random access** *and* the ability to grow without knowing the final size up front.

## The Mechanism and Its Invariant

The structure keeps three pieces of state: a pointer to a heap-allocated buffer, a `length` (elements in use), and a `capacity` (buffer size). The invariant is always `length ≤ capacity`. Reads and writes to `arr[i]` index directly into the buffer — a base-plus-offset address computation — so they are genuinely O(1). The interesting part is `append`.

```ts
class DynamicArray<T> {
  private buf: T[] = new Array(1); // capacity
  private len = 0;                 // length
  push(x: T) {
    if (this.len === this.buf.length) this.grow(); // full → resize
    this.buf[this.len++] = x;
  }
  private grow() {
    const bigger = new Array(this.buf.length * 2); // geometric growth
    for (let i = 0; i < this.len; i++) bigger[i] = this.buf[i]; // O(n) copy
    this.buf = bigger;
  }
}
```

When the buffer is full, `grow` allocates a larger buffer, copies every element over (an O(n) operation), and discards the old one. The critical design choice is that capacity grows **geometrically** — multiply by a constant factor — not arithmetically (adding a fixed slack). This is what makes append cheap on average.

## Complexity and Why Amortized Append Is O(1)

- **Random access `get`/`set`**: O(1) worst case.
- **`push` (append at end)**: O(n) worst case (the resize that copies), but **O(1) amortized**.
- **Insert/delete at an arbitrary index**: O(n) — everything to the right must shift by one to keep the array contiguous. Delete/insert at the very end is the O(1) amortized case.
- **Search (unsorted)**: O(n).

The amortized bound is the subtle claim. With a growth factor of 2, doubling from capacity 1 to `n` triggers resizes at sizes `1, 2, 4, …, n`, copying `1 + 2 + 4 + … + n < 2n` elements *in total* across the whole sequence of `n` pushes. Total work is Θ(n) spread over n operations → **O(1) per push amortized**. Any factor > 1 gives this; a factor of exactly 1 (grow-by-constant) degrades to Θ(n) per push, i.e. Θ(n²) to build the array. See [[amortized-analysis]] for the aggregate, accounting, and potential-method derivations.

## Growth-Factor Trade-offs

The factor is a memory-vs-reallocation-frequency dial:

- **2×** (common, e.g. many `vector` implementations, Java `ArrayList` uses 1.5×): fewer reallocations, but up to ~50% of the buffer can be wasted right after a resize, and the freed old buffer can *never* be reused to satisfy the next allocation from a simple bump allocator (`1+2+4+… < next block`).
- **1.5×** (MSVC `vector`, Java `ArrayList`): more frequent copies but tighter memory, and the growth sequence is friendlier to allocators that can coalesce previously freed blocks into the new request.

Growth is one-directional in most implementations: they rarely shrink on removal (to avoid thrashing), so `capacity` reflects the historical high-water mark. Explicit `reserve`/`ensureCapacity` up front eliminates *all* intermediate resizes when the size is known — the single most effective optimization for hot build loops (see [[07-performance-engineering/algorithmic-optimization|Algorithmic Optimization]]).

## Cache Locality — the Big-O Hides It

Because storage is contiguous, iteration streams linearly through memory, prefetches predictably, and packs many elements per cache line. This is why a dynamic array routinely beats an asymptotically equivalent linked structure by an order of magnitude on real hardware — the constant factor Big-O discards is dominated by memory behavior. See [[07-performance-engineering/data-locality|Data Locality]] and [[07-performance-engineering/cache-friendly-code|Cache-Friendly Code]].

## Dynamic Array vs Linked List

| Operation | Dynamic array | Linked list |
|---|---|---|
| Random access | O(1) | O(n) |
| Append (end) | O(1) amortized | O(1) |
| Insert/delete middle | O(n) (shift) | O(1) *given the node* |
| Memory overhead | low, contiguous | per-node pointer(s) + poor locality |

The linked list's O(1) middle-insert is real only if you *already hold* the node; finding it is O(n). In practice dynamic arrays are the default sequence — locality wins so often that "just use an array" is the correct first instinct.

## Senior Pitfalls

- **Iterator/reference invalidation**: a resize moves the buffer, so any pointer, reference, or index held across a `push` may dangle (a classic C++ `vector` bug). Languages with GC and index-based access hide this, but the semantics still bite when you cache a slice.
- **O(n) `insert(0, …)` in a loop** silently becomes O(n²). Prefer building at the end and reversing, or use a deque.
- **Assuming shrink-to-fit**: memory stays at the high-water mark; a `list` that briefly held 10M items keeps that buffer until explicitly trimmed or replaced.
- **Growth-factor myths**: 2× is *not* universally optimal — see the allocator-reuse argument above.

## See Also

- [[amortized-analysis]]
- [[linked-lists]]
- [[prefix-sums]]
- [[big-o-vs-reality]]
- [[07-performance-engineering/data-locality|Data Locality]]
- [[03-computer-systems/operating-systems/memory-management|Memory Management]]
