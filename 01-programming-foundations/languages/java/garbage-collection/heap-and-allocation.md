# Heap Layout and Allocation

The heap is the region of JVM-managed memory where all objects live, and how it is laid out directly determines how cheap allocation is and how often the collector must run. Java's heap design is built around one empirical observation — most objects die young — and exploits it relentlessly.

## Generational Structure

By default (G1), the heap is logically split into a **young generation** and an **old (tenured) generation**. The young generation is further divided into **Eden** and **two Survivor spaces** (conventionally S0/S1).

```text
Young Generation                       Old (Tenured)
+--------------------+----+----+      +------------------------+
|       Eden         | S0 | S1 |      |   long-lived objects   |
+--------------------+----+----+      +------------------------+
```

New objects are allocated in Eden. When Eden fills, a **minor (young) collection** copies the survivors into one Survivor space; the other survivor space and Eden are then empty (a copying collector — see [[gc-fundamentals]]). Each surviving object carries an **age** (number of collections survived); once it crosses the **tenuring threshold**, it is **promoted** to the old generation. This is why the two survivor spaces exist: collection always copies live objects from "from-space" to "to-space," flipping roles each cycle, which compacts survivors and eliminates fragmentation for free.

## The Weak Generational Hypothesis

The whole scheme rests on the **weak generational hypothesis**: the vast majority of objects become unreachable shortly after allocation. If true, then collecting only the young generation reclaims most garbage while touching only a small slice of the heap — a minor GC is fast precisely because it copies the *few* survivors rather than scanning everything. The old generation is collected far less often, amortizing its higher cost.

## Bump-Pointer Allocation and TLABs

Within Eden, allocation is **bump-the-pointer**: the JVM keeps a pointer to the next free byte and allocating an object is just `ptr += size` plus a bounds check. This is dramatically cheaper than a free-list malloc — close to the cost of a stack push.

The catch is contention: a single bump pointer shared across threads would need an atomic increment per allocation. The JVM avoids this with **TLABs (Thread-Local Allocation Buffers)** — each thread is handed a private chunk of Eden and bumps its pointer there with **no synchronization at all**. Only when a TLAB is exhausted does the thread take a (rare) slow path to grab a new one. TLABs are the reason allocation in Java is essentially free under normal load.

## Allocation Rate as a First-Order Cost

Because minor GCs trigger when Eden fills, **allocation rate** (bytes/sec) is a primary performance lever: doubling allocation roughly doubles GC frequency. A senior pitfall is treating allocation as "free" because each `new` is cheap — the aggregate cost surfaces as GC pressure, not as per-object latency. Reducing churn (object reuse, primitive arrays over boxed collections, avoiding needless intermediate objects) often beats tuning GC flags. See [[07-performance-engineering/data-locality|data locality]] for why allocation patterns also affect cache behavior.

## Metaspace

Class metadata — the runtime representation of classes, methods, constant pools — lives in **Metaspace**, which is **native (off-heap) memory**, not the Java heap. This replaced **PermGen**, removed in Java 8. Metaspace grows dynamically and is bounded by `-XX:MaxMetaspaceSize`. The practical consequence: classloader leaks (e.g., redeploying apps, dynamic proxy generation) manifest as Metaspace exhaustion (`OutOfMemoryError: Metaspace`), wholly separate from heap pressure.

## Escape Analysis: Skipping the Heap

Not every `new` actually allocates on the heap. The JIT performs **escape analysis**: if it can prove an object never escapes the method (or thread) that created it, it can apply **scalar replacement** — exploding the object into its individual fields held in registers/stack slots, so no heap object exists at all. This eliminates the allocation *and* the future GC cost. It is fragile, though: a single store to a static field, return, or passing the reference to an unanalyzable call defeats it. Don't rely on it for correctness or assume it happens — verify with JIT diagnostics. See [[escape-analysis-and-optimizations]].

## Mental Model

Think of the heap as a **conveyor belt**: objects are born in Eden, the cheap and disposable region; the few that survive ride through the survivor spaces, aging; the persistent ones graduate to the old generation. The collector's efficiency comes from rarely needing to look at the tenured end. Compared to JavaScript's heap, which uses a conceptually similar generational design ([[01-programming-foundations/languages/javascript/execution-model/garbage-collector|JS garbage collector]]), Java exposes far more control over sizing and promotion.

## See Also
- [[gc-fundamentals]]
- [[g1-collector]]
- [[escape-analysis-and-optimizations]]
- [[01-programming-foundations/languages/javascript/execution-model/garbage-collector|JS Garbage Collector]]
- [[03-computer-systems/operating-systems/memory-management|OS Memory Management]]
- [[07-performance-engineering/data-locality|Data Locality]]
