# Heap Sort

Heap Sort turns the array into a **binary max-heap**, then repeatedly extracts the maximum by swapping the root to the end and shrinking the heap. It is the go-to when you need `Θ(n log n)` in **every** case with **`O(1)` extra space** — the worst-case guarantee of [[merge-sort]] without merge sort's `Θ(n)` buffer. Its weakness is real-world speed: it thrashes the cache.

## The idea / invariant

A max-heap stored in an array satisfies `a[i] >= a[2i+1]` and `a[i] >= a[2i+2]` — the parent dominates its children, so the maximum is always at index 0. Heap sort has two phases:

1. **Build**: rearrange the array into a max-heap.
2. **Sort**: swap `a[0]` (the max) with the last unsorted slot, shrink the heap by one, and `siftDown` the new root to restore the heap property. Each extraction parks the next-largest element into its final sorted position at the tail.

```
heap:   [9 7 8 3 2 6]      swap 9<->6, shrink
sorted: [6 7 8 3 2 | 9]    siftDown -> heap of size 5
```

## Implementation (in-place, max-heap)

```ts
function heapSort(a: number[]): void {
  const n = a.length;
  // Phase 1: build max-heap bottom-up. Start at last parent, sift each down.
  for (let i = (n >> 1) - 1; i >= 0; i--) siftDown(a, i, n);
  // Phase 2: repeatedly move the max to the end, shrink, re-heapify.
  for (let end = n - 1; end > 0; end--) {
    [a[0], a[end]] = [a[end], a[0]];   // largest -> its final slot
    siftDown(a, 0, end);               // restore heap on a[0..end-1]
  }
}

function siftDown(a: number[], i: number, size: number): void {
  while (true) {
    let largest = i;
    const l = 2 * i + 1, r = 2 * i + 2;
    if (l < size && a[l] > a[largest]) largest = l;
    if (r < size && a[r] > a[largest]) largest = r;
    if (largest === i) return;          // heap property holds -> done
    [a[i], a[largest]] = [a[largest], a[i]];
    i = largest;                        // follow the swapped child down
  }
}
```

## Dry run

Sort `[3, 9, 2, 7]`. Last parent index `= (4>>1)-1 = 1`.

**Build:** siftDown(1): children of `a[1]=9` is `a[3]=7`; 9 ≥ 7, no change. siftDown(0): `a[0]=3` vs children `a[1]=9`, `a[2]=2`; largest is 9 → swap → `[9,3,2,7]`; now index1=3 vs child `a[3]=7` → swap → `[9,7,2,3]`. Max-heap built.

**Sort:**
- end=3: swap a[0]↔a[3] → `[3,7,2,|9]`; siftDown(0,3): 3 vs 7,2 → swap → `[7,3,2,|9]`.
- end=2: swap a[0]↔a[2] → `[2,3,|7,9]`; siftDown(0,2): 2 vs 3 → swap → `[3,2,|7,9]`.
- end=1: swap a[0]↔a[1] → `[2,|3,7,9]`.

Result `[2,3,7,9]`. The sorted region grows from the right, one element per extraction.

## Complexity

**Time — `Θ(n log n)` in best, average, and worst case.** Phase 2 does `n-1` extractions, each an `O(log n)` `siftDown` → `Θ(n log n)`. There is no input that avoids this, so heap sort has **no bad case** (unlike quicksort) — but also no lucky `O(n)` case on sorted data (unlike [[hybrid-sorts|Timsort]]).

**Why the build is `Θ(n)`, not `Θ(n log n)`.** The naive intuition — "`n` elements each sifted `O(log n)` → `n log n`" — over-counts. `siftDown` cost is bounded by a node's **height**, not the tree height. Most nodes are near the leaves with tiny height: `n/2` nodes have height 0, `n/4` height 1, `n/8` height 2, …. Total work `= Σ (n / 2^{h+1}) · h`. The series `Σ h/2^h` converges to 2, so the sum is `Θ(n)`. Building bottom-up is linear; it's the extraction phase that costs the `log n` factor.

**Space — `O(1)`.** Everything happens in the input array; `siftDown` is iterative, so no recursion stack. Truly in-place — heap sort's signature advantage.

**Stable — NO.** Sifting swaps distant elements, and moving the root to the far end scrambles the order of equal keys.

## Big-O vs reality: why it loses in wall-clock

Heap sort and quicksort share `Θ(n log n)` average time, yet quicksort routinely runs 2–3× faster in practice. The reason is **cache locality**, not operation count. `siftDown` hops from index `i` to `2i+1`/`2i+2` — jumps that grow with heap size, so accesses scatter across memory and miss the CPU cache constantly. Quicksort's partition, by contrast, scans memory **sequentially**, which prefetchers love. Big-O ignores the ~100× gap between an L1 hit and a main-memory fetch, so two algorithms with identical asymptotics can have very different clocks. See [[big-o-vs-reality]] — heap sort is the canonical example that Big-O is necessary but not sufficient for predicting speed.

## Variants & trade-offs

- Uses the same [[heaps]] machinery as a [[priority-queues|priority queue]]; heap sort is essentially "insert all, then extract-min/max repeatedly" done in place.
- **Floyd's bottom-up (sift-up) heapify** reduces comparisons in the extraction phase.
- vs merge sort: same worst case but `O(1)` vs `Θ(n)` space, and unstable vs stable.
- vs quicksort: guaranteed no `Θ(n²)`, which is exactly why [[hybrid-sorts|Introsort]] switches *to* heap sort when quicksort recursion gets too deep — heap sort is the safety net.

## When to use / when not

- **Use** when you need a hard `Θ(n log n)` worst-case bound with strictly `O(1)` memory (embedded, kernel, real-time), or as the fallback in a hybrid sort.
- **Don't** pick it as your general array sort when raw speed matters — quicksort/introsort win on cache behavior — or when you need **stability**.

## Interview pitfalls & gotchas

- **Claiming the build is `O(n log n)`** — the tight bound is `O(n)`; be ready to justify with the `Σ h/2^h` argument.
- **Wrong child indices** — 0-based children are `2i+1`, `2i+2`; using `2i`/`2i+1` (1-based) silently corrupts the heap.
- **Sifting past the shrinking boundary** — `siftDown` must respect `size = end`, not the full array, or it drags already-sorted tail elements back in.
- **Confusing sift-down with sift-up** — building bottom-up needs sift-*down* from the last parent; sift-up building is `O(n log n)`.
- **Assuming stability** — it is not stable.

## See Also
- [[heaps]]
- [[priority-queues]]
- [[big-o-vs-reality]]
- [[quick-sort]]
- [[merge-sort]]
- [[hybrid-sorts]]
