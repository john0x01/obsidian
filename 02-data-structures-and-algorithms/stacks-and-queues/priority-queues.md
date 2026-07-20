# Priority Queues

A priority queue is an **abstract data type** (ADT): a collection where each element has a priority and removal always returns the highest- (or lowest-) priority element, regardless of insertion order. It is *not* a data structure — it is an interface, most often implemented with a binary [[heaps|heap]], but the ADT and its implementation should be kept firmly distinct.

## The ADT — Operations

- **insert(x, p)** — add an element with priority `p`.
- **extract-min / extract-max()** — remove and return the highest-priority element.
- **peek / find-min()** — inspect the top without removing.
- **decrease-key(x, p')** — lower an element's key (raise its priority); essential for graph algorithms.
- Optionally **merge/meld** two queues.

Notice what's *absent*: no search, no positional access, no ordered iteration. The contract is narrow on purpose, which is exactly what lets the implementation be fast. A min-PQ and max-PQ differ only by comparator; everything below applies symmetrically.

## Implementation Comparison

| Implementation | insert | extract-min | peek | decrease-key | Notes |
|---|---|---|---|---|---|
| **Binary [[heaps|heap]]** | O(log n) | O(log n) | O(1) | O(log n)† | Array-backed; excellent locality; the default |
| **Sorted array/list** | O(n) | O(1) | O(1) | O(n) | Great if you extract far more than you insert |
| **Unsorted array/list** | O(1) | O(n) | O(n) | O(1) | Great if you insert far more than you extract |
| **Balanced [[binary-search-trees|BST]]** | O(log n) | O(log n) | O(log n)‡ | O(log n) | Overkill unless you also need ordered iteration/range |
| **Binomial / Fibonacci heap** | O(1)amortized | O(log n) | O(1) | O(1) amortized | Fibonacci: theoretical Dijkstra speedup, awful constants |

† Heap decrease-key is O(log n) but needs an index map to locate the element (a raw binary heap can't find an arbitrary key in O(1)). ‡ Peek is O(log n) in a plain BST but O(1) if you cache the min pointer.

The **binary heap wins in practice**: contiguous array storage means cache-friendly sift-up/sift-down, no pointer overhead, and simple code. Fibonacci heaps improve the *asymptotics* of Dijkstra (O(E + V log V)) but their large constants and poor [[07-performance-engineering/data-locality|locality]] mean a binary heap usually runs faster on real graphs. The right choice depends on your insert/extract mix — hence the two degenerate array rows above.

## Building & Bulk Loading

Inserting `n` items one-by-one into a heap is O(n log n), but **heapify** (bottom-up sift-down over an existing array) builds the heap in **O(n)** — a tighter bound than it looks, because most nodes are near the leaves and sift down only a little. When you have all items upfront, heapify then extract; this is the basis of heapsort.

## Applications

- **[[shortest-paths|Dijkstra]] & Prim's MST** — repeatedly extract the closest frontier node and `decrease-key` its neighbours; the PQ is the algorithm's engine. The index-map requirement for decrease-key is the practical wrinkle here (many implementations instead push duplicates and skip stale entries — "lazy deletion").
- **OS / task scheduling** — priority-based ready queues serve the highest-priority runnable task; see [[03-computer-systems/operating-systems/scheduling|scheduling]]. Guard against starvation with aging (bumping long-waiting tasks' priority).
- **Discrete-event simulation** — the event queue is a min-PQ keyed on event time; always process the earliest pending event next.
- **Top-k / streaming** — to keep the `k` largest of a stream, hold a **min-heap of size k**; each element is O(log k), total O(n log k) and O(k) space — far better than sorting everything.
- **Huffman coding** — repeatedly extract the two lowest-frequency nodes and merge; greedy optimality via a min-PQ. Also **A\*** (priority = g + h) and median-maintenance (two heaps).

## Senior Pitfalls

- **Conflating ADT with implementation** — "priority queue" doesn't mean "heap"; pick the implementation from your operation mix. If you never `decrease-key` and extract rarely, an unsorted array can win.
- **decrease-key without an index map** — a bare binary heap can't locate an element in O(1); either maintain a position map or use lazy deletion (push a new entry, discard stale ones on pop). Forgetting this makes Dijkstra secretly O(n) per update.
- **Stability** — heaps are **not stable**; equal-priority elements come out in arbitrary order. If FIFO-within-priority matters, break ties with a monotonically increasing sequence number.
- **Mutating a key in place** — changing an element's priority without re-establishing the heap invariant corrupts the heap; always go through `decrease-key`/`increase-key`.
- **Starvation** — a stream of high-priority work can indefinitely delay low-priority items; add aging in schedulers.
- **Top-k direction** — for the *largest* k use a *min*-heap (so the smallest of the k is cheap to evict); the reversed intuition is a common bug.

## See Also
- [[heaps]] — the standard implementation; heapify, sift-up/down
- [[shortest-paths]] — Dijkstra/Prim built on a priority queue
- [[queues]] — the FIFO ADT this generalizes by priority
- [[monotonic-stack-and-queue]] — the O(n) alternative for sliding-window extrema
- [[03-computer-systems/operating-systems/scheduling|OS Scheduling]] — priority ready queues and aging
