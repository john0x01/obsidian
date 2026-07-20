# GC Tuning and Collector Selection

Tuning GC is not about memorizing flags; it is about matching a collector to a **latency-versus-throughput goal**, sizing the heap sensibly, and then changing one variable at a time against a measured workload. The most common senior mistake is cargo-culting flags from a blog post without an SLO or a GC log to justify them.

## Choosing a Collector

- **G1** — the default since Java 9 and the right first choice for almost all server applications: balanced, region-based, predictable pauses (see [[g1-collector]]).
- **ZGC** — ultra-low, heap-size-independent pauses for latency-critical services and large heaps: `-XX:+UseZGC` (generational by default on modern JDKs; see [[low-pause-collectors]]).
- **Shenandoah** — the other concurrent-compaction option: `-XX:+UseShenandoahGC` (OpenJDK builds only, not Oracle's JDK).
- **Parallel** — the throughput collector: `-XX:+UseParallelGC`. Fully stop-the-world but highest raw throughput; ideal for batch/offline jobs where pauses are irrelevant.
- **Serial** — `-XX:+UseSerialGC`: single-threaded, tiny footprint, good for small heaps and constrained containers.
- **Epsilon** — `-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`: a no-op collector that allocates but never reclaims (the heap fills and the JVM exits). Purely for testing — measuring allocation pressure, isolating GC overhead in benchmarks, or ultra-short-lived jobs.

## Key Flags

- `-Xms` / `-Xmx` — min/max heap; set **equal** in production to avoid runtime resizing pauses.
- `-XX:MaxRAMPercentage` — container-aware sizing; prefer it to a hardcoded `-Xmx` in containers (container support is on by default).
- `-XX:MaxGCPauseMillis` — soft pause target for G1 (default 200) and ZGC.
- `-XX:InitiatingHeapOccupancyPercent` (default 45) — when G1 starts a concurrent marking cycle; `-XX:G1HeapRegionSize` — override region size.
- Avoid micromanaging the young generation (`-XX:NewRatio`, `-XX:SurvivorRatio`, `-XX:MaxTenuringThreshold`) under G1/ZGC — they size young adaptively to meet the pause goal, so fixing these usually fights the collector. They are more legitimate under Parallel.

## Heap Sizing

By default the JVM caps the heap at 25% of physical RAM (`MaxRAMPercentage` default 25.0) — frequently too small for a dedicated service and too large for a packed container. Size deliberately, and remember the heap is not the whole footprint: Metaspace, thread stacks, direct/`ByteBuffer` memory, the JIT code cache, and GC structures (RSets) all live off-heap. Oversizing the heap starves the OS page cache and can *worsen* pauses on non-concurrent collectors; undersizing drives allocation stalls or `OutOfMemoryError`.

## Allocation Rate and Humongous Objects

Allocation rate is a first-order cost: minor collections fire when Eden fills, so halving churn roughly halves young-GC frequency (see [[heap-and-allocation]]). Reducing allocation — object reuse, primitive arrays over boxed collections, fewer throwaway intermediates — usually beats any flag change. Under G1, watch for **humongous allocations** (objects at least half a region); a stream of them fragments the heap and can trigger premature full GCs. Fix by raising the region size or restructuring the large objects.

## Reading GC Logs

Use unified logging (JDK 9+ replaced `-XX:+PrintGCDetails`): `-Xlog:gc*:file=gc.log:time,uptime,level,tags`. Read for: pause durations and their phases; allocation and promotion rates; **evacuation failure / "to-space exhausted"** (G1 under pressure); **Full GC** occurrences (a red flag for G1/ZGC — they should almost never happen); and **allocation stalls** (ZGC/Shenandoah can't keep up). Analyze with JDK Flight Recorder + JDK Mission Control, or offline tools like GCViewer/GCeasy, rather than eyeballing raw text.

## A Tuning Methodology

1. **Define the goal** — a concrete latency SLO (e.g. p99 pause under X ms) *or* a throughput target. Without it, "tuning" is undirected.
2. **Pick the collector** by that goal, not by reputation.
3. **Size the heap**; set `-Xms` = `-Xmx` in production.
4. **Enable GC logging** and run a *representative* load — not a toy benchmark.
5. **Analyze, change one variable, re-measure.** Multi-flag changes make results uninterpretable.
6. **Prefer reducing allocation over flags**, and use a profiler to find where the churn originates (bridge to [[07-performance-engineering/profiling|Profiling]]).

The philosophy: the collector is a servant of your latency/throughput contract. Decide the contract first, measure against it, and touch flags last. See [[07-performance-engineering/garbage-collection|Garbage Collection (perf)]] for the language-agnostic view of the same trade-offs.

## See Also
- [[gc-fundamentals]]
- [[g1-collector]]
- [[low-pause-collectors]]
- [[heap-and-allocation]]
- [[07-performance-engineering/garbage-collection|Garbage Collection (perf)]]
- [[07-performance-engineering/profiling|Profiling]]
