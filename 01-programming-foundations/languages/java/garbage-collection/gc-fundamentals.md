# Garbage Collection Fundamentals

Garbage collection is automatic memory reclamation: the runtime identifies objects the program can no longer reach and frees their memory. Understanding the underlying algorithms — not just which collector to pick — is what lets you reason about pauses, throughput, and pathologies.

## Tracing vs Reference Counting

There are two foundational strategies. **Reference counting** keeps a per-object count of incoming references and frees an object when the count hits zero. It is simple and reclaims promptly, but it cannot collect **cycles** (two objects referencing each other keep each other's count above zero forever) and incurs per-write overhead. The JVM uses **tracing GC** instead: periodically, it starts from a set of known-live roots and traces all reachable objects; whatever it cannot reach is garbage. Tracing handles cycles naturally — reachability, not count, defines liveness — which is the core reason the JVM does not reference-count. (JavaScript engines also moved from naive ref-counting to tracing for the same reason — see [[01-programming-foundations/languages/javascript/execution-model/garbage-collector|JS garbage collector]].)

## GC Roots and Reachability

Tracing begins from **GC roots** — references the program can access without going through the heap:

- Local variables and operand stacks of all live thread stack frames
- Static fields of loaded classes
- JNI references, active monitors, certain JVM-internal structures

An object is **live** if it is reachable by following references from any root. Everything else is dead, regardless of whether the program "intended" to keep it. This is why a forgotten entry in a static `Map` is a classic leak: the static field is a root, so nothing it transitively holds can ever be collected.

## Core Algorithms

- **Mark-sweep**: trace and mark live objects, then sweep the heap freeing unmarked ones. Simple, but leaves **fragmentation** — free space is scattered, and a free-list allocator is needed.
- **Mark-compact**: mark, then slide live objects together to one end, eliminating fragmentation and restoring bump-pointer allocation. Costs an extra pass and pointer fix-ups.
- **Copying**: divide space in two; copy live objects from one half to the other, then flip. Compacts for free and only touches live objects, but halves usable space. Ideal for the young generation, where survivors are few.

Real collectors mix these: young = copying, old = mark-compact or mark-sweep variants.

## Generational Collection

Because most objects die young (the weak generational hypothesis, see [[heap-and-allocation]]), collectors segregate the heap by age and collect the young region frequently and cheaply, the old region rarely. The complication: an old-generation object may reference a young one, so the old generation is also a "root" for young collections — but scanning all of old defeats the purpose.

## Write Barriers, Card Tables, Remembered Sets

The fix is to **track old→young references** as they are created. On every reference store, the JVM runs a tiny **write barrier** that records the location. Implementations:

- **Card table**: the old generation is divided into fixed-size "cards"; a store into a card marks its byte dirty. A young collection scans only dirty cards, not all of old.
- **Remembered sets (RSets)**: per-region structures (used by G1) recording which other regions hold references into a given region, so a region can be collected without scanning the whole heap.

The cost is paid on every pointer write — a deliberate trade of write-time overhead for cheap young collections.

## Stop-the-World, Concurrent, Incremental

- **Stop-the-world (STW)**: all application threads pause while the collector runs. Simple and correct, but pauses scale with the work done.
- **Concurrent**: the collector runs *alongside* application threads, slashing pause times but adding complexity (the heap mutates during collection) and CPU overhead.
- **Incremental**: collection is broken into small chunks interleaved with the application to bound any single pause.

## Tri-Color Marking and the Lost-Object Problem

Concurrent marking uses **tri-color** abstraction: **white** (unvisited, presumed dead), **gray** (visited, children not yet scanned), **black** (fully scanned). Marking finishes when no gray objects remain; surviving whites are collected. The hazard during concurrent marking: if the mutator stores a reference to a white object into a black object **and** removes the last path to it from any gray object, the white object is wrongly collected — the **lost-object problem**. Collectors prevent this with the write barrier, using either **SATB** (snapshot-at-the-beginning — record overwritten references so the original snapshot is preserved; G1's approach) or an **incremental-update** barrier (record new black→white references).

## Safepoints

To trace stacks safely, the JVM must momentarily pause threads at **safepoints** — well-defined points (loop back-edges, method entries/returns) where thread state is consistent and roots are enumerable. The JVM signals a safepoint and waits for every thread to reach one; a thread stuck in a long counted loop without a safepoint poll can delay everyone (**time-to-safepoint** latency). Even "concurrent" collectors need short STW safepoint pauses to start/finish phases.

## See Also
- [[heap-and-allocation]]
- [[g1-collector]]
- [[low-pause-collectors]]
- [[safepoints]]
- [[07-performance-engineering/garbage-collection|Garbage Collection (perf)]]
- [[01-programming-foundations/languages/javascript/execution-model/garbage-collector|JS Garbage Collector]]
