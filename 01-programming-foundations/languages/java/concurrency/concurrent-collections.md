# Concurrent Collections

`java.util.concurrent` ships collections engineered for concurrent access from the ground up, rather than retrofitting thread-safety onto sequential ones. The philosophical break from the old `Collections.synchronizedMap(...)` wrappers is the whole point: those serialize *every* operation behind one lock and still can't make compound actions atomic. The j.u.c collections instead use fine-grained locking or lock-free algorithms, and accept **weaker consistency** in exchange for scalability.

## ConcurrentHashMap: No Whole-Map Lock

`ConcurrentHashMap` (CHM) is the workhorse. Since the Java 8 rewrite it abandoned the old segment/lock-striping design entirely. The table is an array of **bins** (buckets). Reads — `get`, iteration — are **completely lock-free**: nodes hold `volatile` `val`/`next` fields, so a reader sees a consistent, published value without ever acquiring a lock. Writes lock only the **first node of the target bin** (via `synchronized` on that node), so two writes to different bins never contend. Inserting into an *empty* bin skips locking altogether with a single CAS on the array slot. A bin that grows past `TREEIFY_THRESHOLD` (8 nodes, and only once the table has ≥64 slots) converts from a linked list to a **red-black tree**, bounding worst-case lookup at O(log n) and defusing hash-collision DoS.

Two consequences fall out of this design. First, **resize is cooperative**: a thread that finds a resize in progress helps transfer bins, and each migrated bin is marked with a `ForwardingNode` so concurrent readers get redirected to the new table mid-resize. Second, `size()` is approximate — the count is maintained as a striped set of counter cells (the `LongAdder` technique), so under concurrent mutation `size()`/`mappingCount()` is a best-effort snapshot, not a guarantee.

CHM **forbids null keys and values** — unlike `HashMap` — because in a concurrent map `get() == null` would be ambiguous between "absent" and "mapped to null," a distinction you cannot resolve without a lock. Its atomic compound operations (`putIfAbsent`, `compute`, `computeIfAbsent`, `merge`) are the correct way to do check-then-act atomically. The senior pitfall: the function passed to `compute`/`computeIfAbsent` runs **while the bin is locked**, so it must be short and must never modify the *same* map (re-entrant update can deadlock or corrupt bin state).

## Weakly Consistent Iterators

Every j.u.c collection's iterator is **weakly consistent**: it never throws `ConcurrentModificationException`, traverses elements as they existed at some point during the iteration, and *may or may not* reflect mutations made after the iterator was created. This is the deliberate opposite of the **fail-fast** iterators on `ArrayList`/`HashMap`, which track a modification count and throw `CME` the instant the structure changes underneath them. Weak consistency is what makes lock-free reads possible — you cannot offer a snapshot *and* lock-free O(1) reads across an evolving structure, so the API chooses liveness over a globally-consistent view.

## CopyOnWrite, BlockingQueues, and Lock-Free Queues

`CopyOnWriteArrayList` takes the extreme end of that trade-off: every mutation copies the entire backing array under a lock and swaps a single `volatile` reference. Reads and iteration are therefore lock-free and see an immutable **snapshot** (iterators never reflect later writes and never throw CME). This is ideal for read-mostly, rarely-mutated, small collections — the archetype being **listener/observer lists** — and catastrophic for write-heavy use, where each add is O(n) and allocates a fresh array.

`BlockingQueue` is the backbone of producer-consumer pipelines and of the thread pools in [[executors-and-thread-pools]]. Its `put`/`take` **block** when the queue is full/empty, giving natural backpressure. Key implementations differ by internals: `ArrayBlockingQueue` is bounded with a single lock; `LinkedBlockingQueue` uses **two locks** (separate `putLock`/`takeLock`) so a producer and consumer can proceed simultaneously; `SynchronousQueue` has *zero* capacity — each `put` hands directly to a waiting `take` (the direct-handoff queue behind cached thread pools); `PriorityBlockingQueue` and `DelayQueue` add ordering.

`ConcurrentLinkedQueue` is the fully lock-free, unbounded option, implementing the **Michael–Scott** non-blocking algorithm: `offer`/`poll` advance `head`/`tail` pointers via CAS with no locks at all. It shines under high contention where blocking is undesirable, but `size()` is O(n) and inherently unreliable (it walks the list while others mutate).

## The Design Philosophy

`Collections.synchronizedMap` wraps each method in `synchronized(mutex)` — a single global lock that serializes all access and still leaks two problems: compound actions (check-then-act, iterate-then-update) are not atomic and need external synchronization, and iteration must be manually locked or it throws CME. The j.u.c collections dissolve both: atomicity is built into the API (`putIfAbsent`, `compute`), and concurrency comes from bin-level locks or CAS rather than one mutex. The cost you accept is the loss of a strong, snapshot-consistent view — a bargain that is almost always right for shared server-side state.

## See Also

- [[locks-and-atomics]]
- [[executors-and-thread-pools]]
- [[java-memory-model]]
- [[safe-publication]]
- [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]]
- [[03-computer-systems/concurrency-and-parallelism/semaphores-and-monitors|Semaphores and Monitors]]
