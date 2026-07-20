# The G1 Garbage Collector

G1 (Garbage-First), the default collector since Java 9, replaced the monolithic generational heap with a grid of independent **regions**, letting it reclaim the most profitable subset of the heap within a **pause-time target** instead of all-or-nothing. It is the balanced middle ground: far shorter, more predictable pauses than Parallel, without the throughput and complexity cost of the fully concurrent collectors.

## Region-Based Heap

G1 divides the heap into roughly 2048 equal-sized **regions**. Region size is the heap size divided by 2048, rounded to a power of two and clamped to 1–32 MB (overridable with `-XX:G1HeapRegionSize`). Generations are no longer contiguous address ranges; each region is *dynamically* tagged Eden, Survivor, Old, Humongous, or Free.

```text
+----+----+----+----+----+----+----+----+
| E  | O  | S  | F  | O  | H  | H  | E  |
+----+----+----+----+----+----+----+----+
E=Eden  S=Survivor  O=Old  H=Humongous  F=Free
(roles are per-region and change every cycle)
```

"Young generation" is simply the current set of Eden and Survivor regions; its size floats to meet the pause goal. This is the pivotal change: because liveness and reclamation are tracked per region, G1 can collect *some* old regions without touching all of them.

## Young and Mixed Collections

A **young collection** is stop-the-world: it evacuates live objects out of Eden/Survivor regions, ageing survivors and promoting the oldest to Old regions (see [[heap-and-allocation]] for tenuring). A **mixed collection** additionally evacuates a subset of Old regions — the ones a prior marking cycle identified as mostly garbage. There is no separate "young vs old" collector; mixed collections are how G1 reclaims the old generation incrementally rather than in one giant pause.

## Evacuation and Copying

G1 is an **evacuating** collector. Each collection picks a **collection set (CSet)** of regions, copies (evacuates) their live objects into fresh regions, and frees every source region wholesale. Copying compacts for free — no fragmentation, bump-pointer allocation restored — and only touches live objects (cheap when survivors are few). If a target region cannot be found mid-copy ("to-space exhausted" / evacuation failure), G1 must fall back, in the worst case to a full stop-the-world GC (parallelized since JDK 10).

## Remembered Sets and the Write Barrier

To evacuate one region in isolation, G1 must know every reference pointing *into* it from outside the CSet. Per-region **remembered sets (RSets)** record these incoming references. They are maintained by a **post-write barrier**: reference stores that cross regions enqueue dirty cards, which concurrent refinement threads later fold into the target RSet. G1 also runs a **pre-write (SATB) barrier** for marking. This two-part barrier is heavier than Parallel's plain card mark, and RSets cost real memory — the price of region independence.

## SATB Concurrent Marking

To learn which old regions are worth evacuating, G1 runs a concurrent **SATB** (snapshot-at-the-beginning) marking cycle: initial mark (piggybacked on a young pause) to concurrent mark to remark (brief STW) to cleanup. The SATB pre-write barrier records *overwritten* references so the snapshot taken at the start is preserved; anything live at snapshot time is treated as live this cycle, producing some **floating garbage** collected next round. Cleanup ranks old regions by liveness so the next mixed collections evacuate the emptiest first — this is the "garbage-first" heuristic that names the collector.

## The Pause-Time Goal

`-XX:MaxGCPauseMillis` (default 200) is a *soft* target, not a guarantee. G1 keeps a cost-prediction model and sizes the CSet so the predicted pause stays under the goal. Set it too low and G1 shrinks the CSet, causing more frequent collections, higher overhead, and often still missing the target — a classic tuning mistake.

## Humongous Objects

An object at least half a region in size is **humongous** and is allocated directly into one or more contiguous Humongous regions in the old generation, bypassing Eden. This wastes the tail of the last region, fragments the heap, and historically was only reclaimed at the end of a marking cycle (modern G1 eagerly reclaims dead humongous regions during young pauses). The fixes are to raise the region size or avoid outsized arrays.

## Why Regions Beat a Monolithic Old Generation

A single contiguous old generation forces all-or-nothing collection: a full old-gen compaction scans and slides the entire generation, so the pause scales with old-gen size. Regions decouple pause time from heap size — G1 reclaims a bounded CSet incrementally and predictably. The trade is bookkeeping: RSets and the dual barrier cost throughput and memory versus Parallel. G1 deliberately spends that overhead to buy predictable, tunable pauses.

## See Also
- [[gc-fundamentals]]
- [[heap-and-allocation]]
- [[low-pause-collectors]]
- [[gc-tuning-and-selection]]
- [[07-performance-engineering/garbage-collection|Garbage Collection (perf)]]
