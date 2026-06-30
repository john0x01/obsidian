# Garbage Collector

V8's garbage collector reclaims heap memory that the program can no longer reach, so JavaScript can pretend memory is infinite and manual `free` doesn't exist. It exists because tracking object lifetimes by hand is error-prone, and because a *moving* collector can also keep the heap compact, which makes allocation a near-free pointer-bump. The naive picture — "scans the heap and tags unreferenced variables as free" — is a *tracing* collector, but the real engineering is in *not* pausing your program while it does so.

## Reachability and Roots

V8 uses **tracing GC**, not reference counting, so it collects cycles correctly. It starts from a **root set** — the call stack, global object, and engine-internal handles — and transitively marks everything reachable. Anything unmarked is garbage. "Referenced" means *reachable from a root*, which is why a detached object graph that only points at itself still dies. `WeakRef`/`WeakMap`/`WeakSet` entries are deliberately *not* roots: they don't keep their target alive and are cleared when the target is otherwise unreachable.

## The Generational Hypothesis

Empirically, *most objects die young* (the "weak generational hypothesis") — a request handler allocates many short-lived temporaries and a few long-lived ones. V8 exploits this by splitting the heap into a small **young generation** (new space) and a large **old generation**, collecting young far more often and cheaply.

```
Young gen (small, frequent)        Old gen (large, infrequent)
 [ from-space | to-space ]   --->  [ promoted survivors ........ ]
   Scavenger (copying)               Mark-Sweep / Mark-Compact
```

## Young Generation: the Scavenger

New space is two equal **semi-spaces** (from-space, to-space). Allocation is a bump pointer in from-space. On a **minor GC**, the **Scavenger** copies *live* objects into to-space and swaps the roles; dead objects are simply abandoned — no per-object free, so cost is proportional to *survivors*, not garbage. This is Cheney-style semi-space copying (it also compacts as a side effect). Objects that survive one or two scavenges are **promoted** (tenured) into old space. The cost: new space is "half wasted" at any moment, but it's small, so the trade is favourable. Modern V8 also runs scavenging in **parallel** across helper threads.

## Old Generation: Mark-Sweep-Compact

Long-lived objects live in old space, collected by a **major GC**: **mark** all reachable objects, then **sweep** (add dead regions to free lists for reuse) and periodically **compact** (relocate live objects to defragment, fixing the free-list fragmentation that pure sweeping leaves behind). Compaction is what keeps allocation fast despite long-lived churn, but it's the most expensive phase because it moves objects and must update every pointer to them.

## Orinoco: Making Pauses Disappear

A stop-the-world major GC on a big heap is tens of milliseconds — visible jank. **Orinoco** is V8's project to hide that latency through:

- **Incremental marking** — marking is split into small steps interleaved with JS execution, using a tri-colour (white/grey/black) abstraction.
- **Concurrent marking** — most marking runs on background threads *while JS runs*, so the main-thread pause shrinks to a short finalization.
- **Parallel** scavenging and compaction — multiple helper threads share the work during the pause.

### Write Barriers

The hazard with concurrent/incremental marking: mutator code can store a pointer from an already-marked (black) object to an unmarked (white) one, which the collector would then miss and wrongly free. A **write barrier** — a tiny check the compiler emits on every reference store into the heap — records such edges (re-greying the target) so nothing live is lost. Generational GC needs the same mechanism to track old→young pointers (the *remembered set*) without scanning all of old space on a minor GC. Write barriers are the hidden tax that makes generational + concurrent GC correct.

### Idle-Time GC

V8 exposes GC work the embedder can schedule during idle slack. In Chrome, the scheduler hands V8 free time between frames (e.g. the gap before the next `requestAnimationFrame` deadline) to do incremental marking/sweeping when the main thread would otherwise be idle, so collection costs land where the user can't see them.

## Leak Patterns (Logical, Not Engine, Leaks)

GC only frees the *unreachable*. A "leak" in JS is almost always an unintended root:

- **Closures** capturing more than needed — a long-lived callback holding a reference to a large object keeps the whole thing alive.
- **Event listeners / subscriptions** never removed — the emitter roots the handler and its captured scope (React effects, EventTarget, RxJS).
- **Detached DOM** — JS keeps a reference to a node removed from the document, so the node *and its subtree* can't be collected.
- **Unbounded caches / Maps** — entries added, never evicted; use an LRU or `WeakMap` keyed by the object whose lifetime should bound the entry.
- **Module-level / global accumulation** — arrays you push into forever; timers (`setInterval`) whose callback closes over state and is never cleared.

## Heap Snapshots

Chrome DevTools / `--heap-prof` produce **heap snapshots**: the object graph with **retained size** (memory freed if this node were collected) and **retainers** (the path back to a root that keeps it alive). The leak-hunting technique is the **three-snapshot** / comparison method: snapshot, exercise the suspect flow, snapshot again, and inspect objects allocated-but-not-freed — the retainer chain names the rogue root. **Senior pitfalls**: micro-optimizing allocation while a closure roots a megabyte; assuming `delete` or `= null` frees memory immediately (it only removes a reference — collection is async and policy-driven); and reading "shallow size" when "retained size" is what tells you the real cost.

## See Also
- [[runtime]] — the heap this collector manages
- [[v8-architecture]] — engine that hosts the GC
- [[closures]] — captured environments as accidental roots
- [[07-performance-engineering/garbage-collection|Garbage Collection]] — GC algorithms across runtimes
- [[03-computer-systems/operating-systems/memory-management|Memory Management]] — OS-level allocation and paging
- [[07-performance-engineering/frontend-performance|Frontend Performance]] — GC pauses as jank
