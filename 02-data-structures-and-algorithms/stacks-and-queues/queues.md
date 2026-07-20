# Queues

A queue is a **FIFO** (first-in, first-out) collection: you enqueue at the back and dequeue from the front, preserving arrival order. It models any system that must service work fairly in the order it arrived — task schedulers, request buffers, and the frontier of a breadth-first search.

## FIFO Semantics & Core Operations — O(1)

- **enqueue(x)** — add at the rear.
- **dequeue()** — remove and return the front (underflow on empty).
- **peek/front()** — inspect the front without removing.

The invariant is two markers, `head` and `tail`, that only ever move forward (modulo wrap-around). Because both ends are touched in constant time, all core operations are O(1) — *if* the implementation is right, which is where naïve versions fail.

## Implementations

- **Naïve array** — enqueue with `push`, dequeue with `shift`. Correct but **O(n) per dequeue**, because `shift` reindexes the whole array. This is the classic hidden-quadratic bug in a loop.
- **Linked-list-backed** — keep head and tail pointers on a singly [[linked-lists|linked list]]; enqueue at tail, dequeue at head, both worst-case O(1). No resize spikes, but per-node allocation and cache-unfriendly pointer chasing.
- **Circular / ring buffer** — the production default.

### Ring Buffer — Why, and Full-vs-Empty

A ring buffer is a fixed-size array with `head` and `tail` indices that wrap via modulo. It reuses freed slots instead of growing, giving O(1) enqueue/dequeue with contiguous memory, no allocation in steady state, and great cache behaviour — ideal for bounded buffers (audio, network I/O, producer/consumer).

The subtlety: with `head == tail` meaning both **empty** and **full**, you can't distinguish them. Two standard fixes:

1. **Keep a count** (or separate `size`) and test `size == 0` vs `size == capacity`.
2. **Sacrifice one slot** — consider the buffer full when advancing `tail` would collide with `head`, so a capacity-`n` array holds `n − 1` elements.

```ts
class RingBuffer<T> {
  private a: (T | undefined)[]; private head = 0; private size = 0;
  constructor(private cap: number) { this.a = new Array(cap); }
  enqueue(x: T): boolean {
    if (this.size === this.cap) return false;      // full
    this.a[(this.head + this.size) % this.cap] = x;
    this.size++; return true;
  }
  dequeue(): T | undefined {
    if (this.size === 0) return undefined;         // empty
    const x = this.a[this.head];
    this.head = (this.head + 1) % this.cap; this.size--; return x;
  }
}
```

## Variants

- **Deque (double-ended queue)** — push/pop at *both* ends in O(1). A doubly linked list or a growable ring buffer backs it. A deque subsumes both stack and queue and is the engine behind the [[monotonic-stack-and-queue|monotonic deque]] used for sliding-window extrema.
- **Two-stacks queue** — implement a FIFO queue from two LIFO [[stacks]]: push onto an `in` stack; to dequeue, if `out` is empty, pour everything from `in` into `out` (reversing order), then pop `out`. Each element is moved at most twice, so dequeue is **amortized O(1)** even though a single transfer is O(n). A clean [[amortized-analysis|amortized-analysis]] example — the expensive transfer pays for the many cheap pops that follow.
- **Priority queue** — not FIFO; serves by priority, not arrival. Different ADT, different implementation (a heap). See [[priority-queues]].

## Complexity

enqueue, dequeue, peek: **O(1)** — worst-case for linked and ring-buffer implementations; amortized O(1) for the two-stacks queue and for a resizing array queue under doubling. Space O(n) (O(capacity) for a fixed ring buffer). The naïve `shift`-based queue is O(n) per dequeue — avoid it.

## Applications

- **BFS** — the queue *is* the frontier; dequeuing in arrival order guarantees nodes are visited in nondecreasing distance from the source, which is what makes BFS find shortest paths in unweighted [[graphs|graphs]]. See [[graph-traversal]].
- **Scheduling** — run queues, thread pools, and OS ready queues serve tasks fairly; see [[03-computer-systems/operating-systems/scheduling|OS scheduling]].
- **Buffering / backpressure** — bounded queues (ring buffers) decouple fast producers from slow consumers and provide natural backpressure when full.

## Senior Pitfalls

- **`Array.shift` in a hot loop** — silently O(n); the top cause of a "fast" BFS running quadratically. Use a ring buffer, a linked list, or an index-based head pointer that you advance instead of shifting.
- **Ring-buffer full/empty ambiguity** — pick the count or wasted-slot convention deliberately; getting it wrong drops or overwrites data.
- **Unbounded growth** — an unbounded queue with a slow consumer is a memory leak / OOM waiting to happen; bound it and define an overflow policy (block, drop-oldest, reject).
- **Confusing queue with priority queue** — "queue" in scheduling contexts often means priority-ordered, not FIFO.

## See Also
- [[stacks]] — the LIFO counterpart; two of them build a queue
- [[monotonic-stack-and-queue]] — a deque with an ordering invariant
- [[graph-traversal]] — BFS uses a queue as its frontier
- [[priority-queues]] — priority-ordered, not arrival-ordered
- [[amortized-analysis]] — the two-stacks queue analysis
- [[03-computer-systems/operating-systems/scheduling|OS Scheduling]] — run/ready queues
