# LRU Cache

An LRU (Least Recently Used) cache is a fixed-capacity key–value store that, when full, evicts the entry that has gone longest without being touched. It solves the bounded-memory caching problem: keep the *hot* working set resident and discard the cold tail, using recency of access as a cheap proxy for future usefulness (the temporal-locality assumption).

## The Idea: Two Structures, One Invariant

The classic interview design pairs two data structures because neither alone gives O(1) for both operations:

- A **hash map** from key → node, for O(1) *lookup* ("is this key present, and where is its node?"). A [[linked-lists|linked list]] alone would force an O(n) scan to find a key.
- A **doubly-linked list** of nodes ordered by recency, for O(1) *reordering and eviction*. A hash map alone has no notion of order, so it cannot tell you which entry is coldest.

**The recency invariant:** the list is kept in most-recently-used → least-recently-used order. The head is the freshest entry; the tail is the eviction candidate. Every access re-establishes this invariant by moving the touched node to the head.

Why *doubly*-linked and why store the node in the map? To move a node to the head in O(1) you must splice it out of its current position, and unlinking a node without walking to it requires its `prev` pointer. A singly-linked list would need an O(n) search for the predecessor. So the map hands you the node directly, and the two `prev`/`next` pointers let you detach it in constant time.

```ts
// map: Map<K, Node>;  list sentinels: head (MRU side) <-> tail (LRU side)
function get(key: K): V | undefined {
  const node = map.get(key);
  if (!node) return undefined;
  moveToFront(node);          // unlink + insert after head — O(1)
  return node.value;
}
function put(key: K, value: V): void {
  if (map.has(key)) { const n = map.get(key)!; n.value = value; moveToFront(n); return; }
  if (map.size === capacity) {
    const lru = tail.prev;    // node just before the tail sentinel
    unlink(lru); map.delete(lru.key);   // evict coldest — O(1)
  }
  const n = new Node(key, value); insertAfterHead(n); map.set(key, n);
}
```

Sentinel head/tail nodes remove null-checking on the ends and make every splice branch-free — a standard trick that eliminates the fiddliest bugs in the implementation.

## Complexity

- **`get`, `put`: O(1) worst case** for the list surgery, **O(1) expected** overall because they ride on the hash map (see [[hash-tables]]) — so the true statement is *amortized/expected O(1)*, degrading to O(n) only under a pathological hash. The list operations themselves are true constant-time, not amortized.
- **Space: O(capacity)** — one map entry plus one list node per resident key, roughly 2–3 pointers of overhead per entry beyond the payload.

The design's elegance is that recency bookkeeping adds zero asymptotic cost: reordering is folded into the same O(1) pointer writes that a plain hash map would already do.

## Variants & Trade-offs

- **LFU (Least Frequently Used):** evicts by *access count* rather than recency; better when popularity is stable, but it clings to entries that were hot once and never adapts to phase changes. LRU is scan-resistant to nothing, LFU is scan-resistant to bursts — real caches (e.g. `W-TinyLFU` in Caffeine) blend both.
- **Java `LinkedHashMap(accessOrder=true)`:** the JDK bakes exactly this map+list into one structure; override `removeEldestEntry` and you have an LRU in a few lines — the standard production shortcut.
- **CLOCK / second-chance:** an *approximation* used by OS page replacement (see [[03-computer-systems/operating-systems/memory-management|Memory Management]]). Instead of a linked list it keeps a circular buffer with one *reference bit* per slot; a hand sweeps, clearing set bits and evicting the first slot whose bit is already 0. It avoids the per-access pointer writes (a big deal for cache-line contention) at the cost of only approximating true LRU.
- **Segmented / 2Q:** split into probationary and protected segments to resist one-shot scans flooding out the hot set — a known LRU weakness.

## When To Use It & Pitfalls

Use LRU for page and object caches, DB buffer pools, DNS/CDN edge caches, and as the eviction policy behind bounded [[07-performance-engineering/memoization|Memoization]] tables where an unbounded memo would leak memory.

- **The list writes hurt under concurrency:** every `get` mutates shared list pointers, so a naïve lock serializes reads. Production caches batch or defer the reordering (record accesses in a ring buffer, replay them under a tryLock) precisely to avoid this.
- **LRU is not scan-resistant:** one sequential pass over more-than-capacity cold keys evicts your entire hot set. Admission policies (TinyLFU) or segmentation fix this.
- **Don't forget to delete from *both* structures on eviction** — a node unlinked from the list but left in the map is a memory leak and a correctness bug.

## See Also
- [[hash-maps-and-sets]] — the O(1) lookup half of the design
- [[linked-lists]] — the doubly-linked recency list and O(1) splicing
- [[hash-tables]] — why the lookup is *expected* O(1), not guaranteed
- [[07-performance-engineering/memoization|Memoization]] — bounding a memo table with an eviction policy
- [[03-computer-systems/operating-systems/memory-management|Memory Management]] — CLOCK/second-chance page replacement
