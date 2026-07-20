# Heaps

A **heap** is a complete binary tree that maintains a partial order — every parent dominates its children — giving O(1) access to the minimum (or maximum) element and O(log n) insertion and removal. It is the workhorse implementation of a [[priority-queues|priority queue]].

## The Idea and Invariant

A binary heap keeps two invariants at once:

- **Shape:** the tree is *complete* — every level is full except possibly the last, which fills left to right. This lets us store it in a flat array with no pointers.
- **Heap-order:** in a min-heap, `key(parent) ≤ key(child)` for every node (max-heap flips the comparison). Siblings are *not* ordered relative to each other — this is a partial order, which is exactly why a heap is cheaper to maintain than a fully sorted structure or a [[binary-search-trees|BST]].

Because the tree is complete, the array layout is implicit. For a 0-indexed array:

```ts
const parent = (i: number) => (i - 1) >> 1;
const left   = (i: number) => 2 * i + 1;
const right  = (i: number) => 2 * i + 2;
```

No child pointers, no allocation per node, and children sit near each other in memory — good for [[07-performance-engineering/data-locality|data locality]].

## Sift-Up and Sift-Down

Two primitives repair the order property after a local violation:

- **sift-up** (bubble up): a node too small for its position swaps with its parent, repeatedly, until the order holds. Used after appending.
- **sift-down** (sink/heapify): a node too large sinks by swapping with its *smaller* child until it settles. Used after replacing the root.

Both walk at most the tree height, `⌊log₂ n⌋`, so they are O(log n).

```ts
// sift-down for a min-heap
function siftDown(a: number[], i: number, n: number) {
  while (true) {
    let m = i, l = 2 * i + 1, r = 2 * i + 2;
    if (l < n && a[l] < a[m]) m = l;
    if (r < n && a[r] < a[m]) m = r;
    if (m === i) break;
    [a[i], a[m]] = [a[m], a[i]];
    i = m;
  }
}
```

- **push:** append at the end, sift-up — O(log n).
- **pop (extract-min):** save the root, move the last element to the root, shrink, sift-down — O(log n).
- **peek:** O(1).

## Build-Heap in O(n)

Given an arbitrary array, calling sift-down on every non-leaf node from the last parent (`⌊n/2⌋ − 1`) down to index 0 produces a valid heap — Floyd's method. The naive analysis says "n sift-downs × O(log n) = O(n log n)," but the **tight bound is O(n)**, and the reason is worth internalizing.

Work per node is proportional to its *height*, not the tree's height. In a complete tree, most nodes are near the bottom, where height is small: roughly `n/2` nodes are leaves (height 0, zero work), `n/4` have height 1, `n/8` height 2, and so on. The total is

```
Σ (n / 2^{h+1}) · h  =  n · Σ h/2^{h+1}  →  n · 1  =  O(n)
```

since `Σ h/2^h` converges to 2. The cheap many outweigh the expensive few. Note the contrast: inserting n elements one at a time is `Σ log i = O(n log n)`, genuinely slower. This is a classic [[amortized-analysis|amortized]]/aggregate argument — see also [[big-o-vs-reality]].

## Heapsort

Build a max-heap in O(n), then repeatedly swap the root to the shrinking tail and sift-down the new root. n extractions × O(log n) gives **O(n log n) worst *and* average** — heapsort has no pathological input like quicksort's bad pivot, and it is **in-place** (O(1) extra space).

Two senior caveats: heapsort is **not stable** (swaps reorder equal keys arbitrarily), and despite matching quicksort's asymptotics it is usually *slower in practice* — its access pattern jumps across the array, thrashing the cache, so it loses to a cache-friendly quicksort. See [[sorting-algorithms]].

## Variants and Trade-offs

- **d-ary heap:** each node has d children, so the tree is shallower — height `log_d n`. This makes sift-up (and *decrease-key*) faster, but sift-down/pop must scan d children per level, costing `O(d·log_d n)`. Choosing d ≈ E/V speeds up Dijkstra on dense graphs, and the flatter layout is more cache-friendly.
- **Fibonacci heap:** offers *amortized* O(1) insert, merge, and **decrease-key**, with O(log n) amortized extract-min. This lowers Dijkstra/Prim to `O(E + V log V)` on paper (see [[shortest-paths]], [[minimum-spanning-tree]]). In practice its large constants and pointer-chasing rarely beat a plain binary heap; a **pairing heap** is the usual simpler stand-in.

## When to Use It

Reach for a heap whenever you need repeated access to the extremum of a changing set: schedulers ([[03-computer-systems/operating-systems/scheduling|OS scheduling]]), Dijkstra/Prim, top-k / streaming median (two heaps), event simulation, and Huffman coding. If you need arbitrary-key lookup or ordered iteration, a heap is the wrong tool — use a balanced BST.

**Pitfalls:** a heap does *not* support O(log n) search or delete of an arbitrary key without an external index; `decrease-key` needs a handle mapping keys to positions; and never assume sorted iteration — only the root is ordered.

## See Also
- [[priority-queues]] — the abstract type a heap implements
- [[shortest-paths]] — Dijkstra/Prim consume a heap
- [[sorting-algorithms]] — heapsort in context
- [[binary-trees]] — the complete-tree shape underneath
- [[03-computer-systems/operating-systems/scheduling|OS Scheduling]] — priority queues in the kernel
