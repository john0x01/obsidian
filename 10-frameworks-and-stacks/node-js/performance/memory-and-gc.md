# Memory and GC

Node memory problems are rarely "we allocate too much" — V8's generational GC is extremely good at churning short-lived garbage. The problems that page you are **leaks** (objects that should die but are kept reachable) and **fragmentation/limits** (the old space hits `--max-old-space-size` and the process OOM-crashes or thrashes in GC). Mastery here is about reading the right number and finding *what keeps an object alive*.

## The V8 heap in Node

V8 splits the managed heap into a **young generation** (new space, a small semi-space scavenger using copying GC) and an **old generation** (old space, collected by a mark-sweep-compact major GC, mostly concurrent and incremental in modern V8). The hypothesis is generational: most objects die young, so the cheap scavenger handles them; survivors get *promoted* to old space. There are also dedicated spaces (code, large-object space for allocations over the page size, etc.). Crucially, the old-space ceiling is what `--max-old-space-size` (MB) caps. The default is sized off available RAM, but in a container with a memory limit the heuristic can guess wrong — **set it explicitly to ~75-80% of the container limit**, leaving headroom for non-heap memory (see RSS below) or you will OOM-kill before V8 even decides to GC harder.

## RSS vs heapUsed vs external

`process.memoryUsage()` returns several numbers that mean very different things, and conflating them is the classic mistake:

```
RSS ─────────────────────────────────────────────┐  resident set: all physical pages
 ├─ heapTotal   V8-reserved heap                  │  (heap + native + stacks + code)
 │   └─ heapUsed   live JS objects in that heap   │
 ├─ external    C++ objects bound to JS (Buffers' backing store, etc.)
 └─ arrayBuffers  subset of external: ArrayBuffer/Buffer memory
```

- **heapUsed** climbing without bound ⇒ a JS-object leak (closures, caches, listeners).
- **external/arrayBuffers** climbing ⇒ Buffers/typed arrays not being released (common with streaming or zlib pools); these live *outside* the V8 heap and won't show in a heap snapshot's retained size the same way.
- **RSS** climbing while heapUsed is flat ⇒ native leak, fragmentation, or glibc/jemalloc not returning freed pages to the OS. RSS is also "sticky" — V8 may free heap internally without shrinking RSS, which alarms people who watch only RSS.

## Capturing and comparing heap snapshots

A heap snapshot is a full graph of every reachable object with **shallow size** (the object itself) and **retained size** (everything that would be freed if this object died). Capture via DevTools (Memory tab), `node --heapsnapshot-signal=SIGUSR2` (write on signal — safe for prod), `v8.writeHeapSnapshot()`, or the inspector `HeapProfiler` domain.

The technique that actually finds leaks is the **three-snapshot diff**: snapshot, run the suspected leaky workload N times, snapshot, repeat, then in DevTools use **Comparison** view to see what grew. Sort by **Delta** and **Retained Size**. The smoking gun is a **constructor whose object count grows by ~N each cycle**. Then open one instance and read its **Retainers** path (the chain of references keeping it alive) — that path *is* the bug.

## Where Node leaks actually come from

- **Closures over large scope.** A closure retains its entire enclosing variable environment, not just the variables it names. A long-lived callback (a cache entry, an event handler) that closes over a big buffer pins that buffer forever.
- **Module-level / global caches without eviction.** `const cache = new Map()` at module scope lives for the process lifetime. Unbounded memoization, request-keyed caches, and "I'll just store it globally" are the #1 source. Use bounded LRU or `WeakMap`/`WeakRef` when keyed by object identity.
- **EventEmitter listeners.** Adding a listener on a long-lived emitter (a singleton, `process`, a shared stream) in a per-request path and never calling `removeListener` accumulates closures. Node's `MaxListenersExceededWarning` (default 10) is your early-warning system — treat it as a leak signal, do not paper over it by raising the limit.
- **Timers and unsettled promises.** A `setInterval` never `clearInterval`'d, or a promise that never resolves, retains its callbacks and captured scope.

## GC tuning flags (use sparingly)

The defaults are good; tuning is a last resort after you've cut allocation. Useful knobs:

- `--max-old-space-size=<MB>` — the one you will actually set, to match container limits.
- `--max-semi-space-size=<MB>` — enlarges new space so the scavenger runs less often; can reduce promotion of medium-lived objects and cut major-GC pressure for allocation-heavy services. Bench it; bigger is not free.
- `--expose-gc` enables `global.gc()` — *for tests and diagnostics only* (e.g. force GC before/after a snapshot to drop floating garbage). Never call it in production hot paths; you are usually slower than V8's scheduler.
- `--trace-gc` / `--trace-gc-verbose` log every GC with type, pause time, and space sizes — the cheapest way to confirm "are we GC-thrashing?". Frequent long *major* (mark-sweep) pauses near the heap limit means you're memory-bound, not slow.

## Senior pitfalls and philosophy

Watch **heapUsed and external separately** — a Buffer leak hides from heapUsed. A flat-then-cliff RSS graph is often the OOM-killer, not a slow leak; correlate with `--max-old-space-size`. Don't chase "memory grows then stabilizes" — that's usually V8 sizing its heap to the workload, which is healthy. The real test of a leak is **monotonic growth across many steady-state cycles**, which is exactly what the three-snapshot diff isolates. Finally, remember the goal is *reachability*: GC frees the unreachable, so every leak is a "who still points at this?" question, and the retainer path always answers it.

## See Also
- [[profiling-and-diagnostics]]
- [[event-loop-monitoring]]
- [[01-programming-foundations/languages/javascript/execution-model/garbage-collector|JS Garbage Collector]]
- [[07-performance-engineering/garbage-collection|Garbage Collection]]
- [[07-performance-engineering/memory-bound-optimization|Memory-Bound Optimization]]
- [[01-programming-foundations/languages/javascript/language-semantics/scopes-and-closures|Scopes and Closures]]
