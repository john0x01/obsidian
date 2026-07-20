# Low-Pause Collectors: ZGC and Shenandoah

ZGC and Shenandoah are **concurrent-compaction** collectors: they relocate live objects while application threads keep running, keeping pauses in the sub-millisecond to low-millisecond range and — the defining property — **independent of heap size and live-set size**. They exist for latency-sensitive services where even G1's ~200 ms target is too coarse.

## The Hard Problem: Moving Objects Concurrently

Marking concurrently is well understood (see the tri-colour discussion in [[gc-fundamentals]]). The genuinely hard part is **relocation/compaction** while the mutator runs: a thread may hold, dereference, or store through a reference to an object that the collector is simultaneously moving. Both collectors solve this with a **barrier on reference loads** that transparently redirects access to the object's current location. They differ in how that redirection is encoded.

## ZGC: Colored Pointers and Load Barriers

ZGC (production since JDK 15, JEP 377) stores GC metadata *inside the 64-bit reference itself* — **colored pointers**, with bits indicating marking/relocation state. Multiple virtual addresses are mapped to the same physical memory so the colored bits don't disturb dereferencing. Being pointer-based, it is 64-bit only.

Every load of a heap reference runs a **load barrier**: it inspects the color bits, and if the pointer is "bad" (points at an object already relocated, or not yet processed this cycle) it takes a slow path that consults the forwarding information, then **self-heals** by rewriting the reference in place so the check passes cheaply next time. Relocation therefore proceeds concurrently; the barrier lazily fixes up references as the program touches them.

**Generational ZGC** was added as opt-in in JDK 21 (JEP 439), made the *default* ZGC mode in JDK 23 (JEP 474), and the old single-generation mode was removed in JDK 24 (JEP 490). So on Java 25, `-XX:+UseZGC` gives you generational ZGC with no separate flag. Splitting into young and old generations exploits the weak generational hypothesis, cutting the CPU and heap overhead of the original design, which had to process the whole heap every cycle. Generational ZGC also tracks cross-generational references, using store-side barriers/remembered-set structures in addition to the load barrier. (I'm describing the mechanism; the exact colored-pointer bit layout has changed across versions and I won't assert specific bit positions.)

## Shenandoah: Forwarding Pointers and Load-Reference Barriers

Shenandoah (from Red Hat; production since JDK 15) encodes redirection differently. Each object carries a **Brooks-style forwarding pointer** — a word that normally points to the object itself, but during evacuation points to the object's new copy. All access goes *through* the forwarding pointer, so every thread sees one canonical copy. A **load-reference barrier (LRB)** on reference loads ensures you obtain the to-space copy and can self-heal the reference. Newer Shenandoah reworked the early write-barrier-heavy scheme toward load-reference barriers and folded the indirection into the object header to shrink footprint; it is non-generational by default (a generational mode has been experimental). Note that Shenandoah ships in OpenJDK builds but not Oracle's JDK, so availability depends on your distribution.

## What's Still Stop-the-World

Neither is fully pauseless. Short STW pauses remain for phase transitions and root scanning, but recent versions scan thread stacks concurrently, driving pauses down to roughly tens of microseconds to a couple of milliseconds — and, critically, that pause does *not* grow as the heap grows. This is the whole point: on a 100 GB heap, G1's pause scales with the work; ZGC's does not.

## The Latency-versus-Throughput Trade

Concurrency is not free, and treating these as strictly better than G1 is a senior misconception:

- **Barriers tax every access.** A load (and, for generational/store tracking, some stores) runs barrier code, lowering peak throughput versus G1 or Parallel.
- **Floating garbage and headroom.** Concurrent collection needs spare heap to allocate into while it works. If allocation outpaces the collector, you hit **allocation stalls** — application threads blocked waiting for memory, the low-pause analog of a long STW pause.
- **More CPU and footprint.** GC threads consume cores concurrently, and per-object metadata (forwarding word, extra mappings) raises overhead.

The selection rule follows directly: Parallel for maximum throughput on batch work where pauses don't matter, G1 as the balanced default, and ZGC/Shenandoah when p99/p999 tail latency and large heaps dominate. See [[gc-tuning-and-selection]] for choosing and measuring.

## See Also
- [[gc-fundamentals]]
- [[g1-collector]]
- [[gc-tuning-and-selection]]
- [[heap-and-allocation]]
- [[07-performance-engineering/latency-vs-throughput|Latency vs Throughput]]
- [[07-performance-engineering/tail-latency|Tail Latency]]
