# Linked Lists

A linked list stores a sequence as a chain of independently allocated **nodes**, each holding a payload and one or more pointers to its neighbours. It trades the contiguous memory (and O(1) random access) of an array for **O(1) structural edits at a known position** — splicing a node in or out without shifting anything.

## The Idea & Node Layout

Each node is a heap object; the list is a reference to the first node (the **head**), and the last node's `next` is null (or, in a ring, points back to the head).

```ts
class Node<T> {
  value: T;
  next: Node<T> | null = null;
  prev?: Node<T> | null; // only in a doubly linked list
  constructor(v: T) { this.value = v; }
}
```

The invariant is simple but load-bearing: following `next` from the head reaches every element exactly once and terminates. Every mutation must re-establish it, which is where the bugs live — a dropped or duplicated pointer silently corrupts the whole chain.

## Variants

- **Singly linked** — one `next` pointer. Minimal memory, but you can only walk forward, and deleting a node requires its *predecessor*.
- **Doubly linked** — `next` and `prev`. Enables O(1) deletion given only a node reference and backward traversal, at the cost of one extra pointer per node and two pointers to fix per edit. This is what backs most production deques and [[lru-cache|LRU caches]].
- **Circular** — the tail links back to the head (singly or doubly). Natural for round-robin scheduling and ring buffers; you must track a count or a sentinel to know when you've looped.

## Core Operations & Complexity

| Operation | Singly | Doubly |
|---|---|---|
| Insert/delete at head | O(1) | O(1) |
| Insert at tail (tail ptr kept) | O(1) | O(1) |
| Delete at tail | O(n) | O(1) |
| Insert/delete **given the node** | O(1)* | O(1) |
| Access / search by value or index | O(n) | O(n) |

\*Singly-linked "delete given a node" needs the predecessor. The classic trick — copy the successor's value into the node and unlink the successor — is O(1) but **cannot delete the tail** and breaks identity if other references point at the copied-over node.

The headline win is the O(1) splice: given a pointer to where you're editing, insertion and deletion are constant time and touch no other element. Arrays cannot match this — a mid-array insert is O(n) because of the shift. The headline loss is **no random access**: reaching index *k* is O(k), and there is no binary search because you can't jump to the middle.

## The Cache Truth the Big-O Hides

Asymptotics say list and array traversal are both O(n), but on real hardware the array wins decisively. Array elements are **contiguous**, so one cache line prefetches the next several; list nodes are scattered across the heap, so each `next` dereference is a likely cache miss — **pointer chasing** that stalls the CPU on memory latency. A "slower-looking" contiguous pass often beats an O(n) list walk by an order of magnitude. This is the core lesson of [[07-performance-engineering/data-locality|data locality]]: prefer arrays unless you genuinely need O(1) splicing at held positions. See also [[07-performance-engineering/cache-friendly-code|cache-friendly code]].

## Sentinel / Dummy-Head Nodes

A sentinel (dummy) node is a non-data node placed before the real head (and often after the tail). It eliminates the special-casing of empty-list and head-of-list edits: with a dummy head there is *always* a predecessor, so "insert before the first element" runs the same code path as any other insert. This deletes the most error-prone branches in list code and is standard in mature implementations — a doubly linked list with two sentinels never needs null checks inside the hot path.

## Array vs List — When to Use Which

Reach for a linked list when you do many insertions/deletions **at positions you already hold a reference to** (splice-heavy workloads, intrusive kernel lists, graph adjacency lists, the eviction list of an [[lru-cache|LRU cache]]) and rarely index. Reach for a [[dynamic-arrays|dynamic array]] for essentially everything else: iteration, indexing, sorting, binary search, and any cache-sensitive path. Dynamic arrays give amortized O(1) append (see [[amortized-analysis]]) with far better constants. In practice the array is the default and the list is the specialist.

## Senior Pitfalls

- **Lost-pointer corruption**: reorder assignments so you never drop the only reference to the rest of the chain (save `next` before rewiring — see [[in-place-reversal]]).
- **Off-by-one on head/tail**: forgetting to update the `tail` pointer or the `prev` of the new head; sentinels prevent most of these.
- **Cycles**: an accidental self-link turns traversal into an infinite loop — detect with [[fast-slow-pointers|Floyd's algorithm]].
- **Memory overhead & GC pressure**: one to three pointers plus per-object allocation headers per element; in GC'd runtimes many small nodes stress the collector.

## See Also
- [[dynamic-arrays]] — the contiguous alternative and default choice
- [[fast-slow-pointers]] — cycle detection and midpoint in one pass
- [[in-place-reversal]] — pointer rewiring with an invariant
- [[stacks]] · [[queues]] — classic linked-list-backed ADTs
- [[07-performance-engineering/data-locality|Data Locality]] — why arrays beat lists in cache
- [[lru-cache|LRU Cache]] — a canonical doubly-linked-list application
