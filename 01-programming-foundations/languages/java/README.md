# Java

A deep track on the Java language, the JVM, its memory model, GC, and concurrency — aimed at mastery for an engineer coming from JavaScript. Internals, mental models, and design philosophy, not syntax. Scoped to the language + runtime core; ecosystem frameworks (Spring, Jakarta EE, Maven/Gradle) are a deliberate follow-up, not covered here.

[← Back to Programming Foundations](../../README.md)

## Orientation (start here)
- [Java vs JavaScript](philosophy/java-vs-javascript.md) — map your JS mental models onto Java, and where they diverge
- [Design Philosophy](philosophy/design-philosophy.md) — WORA, static typing, backward compatibility, "boring is a feature"
- [Java Evolution & JEPs](philosophy/java-evolution-and-jeps.md) — the 6-month cadence, LTS, preview features
- [The JVM as a Platform](philosophy/the-jvm-as-a-platform.md) — bytecode as a shared target; Kotlin/Scala/Clojure

## Language & Type System
- [Type System & Conversions](language-semantics/type-system-and-conversions.md)
- [Generics & Erasure](language-semantics/generics-and-erasure.md)
- [Variance & Wildcards](language-semantics/variance-and-wildcards.md)
- [Records & Sealed Types](language-semantics/records-and-sealed-types.md)
- [Pattern Matching & switch](language-semantics/pattern-matching-and-switch.md)
- [Enums & Annotations](language-semantics/enums-and-annotations.md)
- [Lambdas & Functional Interfaces](language-semantics/lambdas-and-functional-interfaces.md)
- [Equality & Contracts](language-semantics/equality-and-contracts.md) — `equals`/`hashCode`/`compareTo`
- [Nullability & Optional](language-semantics/nullability-and-optional.md)
- [Exceptions & Error Handling](language-semantics/exceptions-and-error-handling.md) — checked vs unchecked

## The JVM
- [JVM Architecture](jvm/jvm-architecture.md)
- [Class Loading](jvm/class-loading.md)
- [Bytecode & Class Files](jvm/bytecode-and-class-files.md)
- [Execution Engine & JIT](jvm/execution-engine-and-jit.md) — interpreter, C1/C2, tiered, OSR, deopt
- [Escape Analysis & Optimizations](jvm/escape-analysis-and-optimizations.md)
- [Safepoints](jvm/safepoints.md) — STW coordination, time-to-safepoint, profiler bias

## Java Memory Model
- [The Java Memory Model](memory-model/java-memory-model.md) — happens-before, data races
- [`volatile` & `synchronized`](memory-model/volatile-and-synchronized.md)
- [Final-Field Semantics](memory-model/final-field-semantics.md)
- [Safe Publication](memory-model/safe-publication.md)

## Garbage Collection
- [GC Fundamentals](garbage-collection/gc-fundamentals.md) — generational model, roots, STW
- [Heap & Allocation](garbage-collection/heap-and-allocation.md)
- [The G1 Collector](garbage-collection/g1-collector.md) — the region-based default
- [Low-Pause Collectors](garbage-collection/low-pause-collectors.md) — ZGC & Shenandoah
- [GC Tuning & Selection](garbage-collection/gc-tuning-and-selection.md) — choosing a collector, Epsilon

## Concurrency
- [Threads & the OS](concurrency/threads-and-the-os.md)
- [Executors & Thread Pools](concurrency/executors-and-thread-pools.md)
- [Fork/Join & Parallel Streams](concurrency/fork-join-and-parallel-streams.md)
- [Locks & Atomics](concurrency/locks-and-atomics.md) — `synchronized`, AQS, CAS
- [Concurrent Collections](concurrency/concurrent-collections.md) — `ConcurrentHashMap` & friends
- [Virtual Threads](concurrency/virtual-threads.md) — Project Loom; contrast with the JS event loop
- [Structured Concurrency](concurrency/structured-concurrency.md) — `StructuredTaskScope`, scoped values
- [CompletableFuture & Async](concurrency/completablefuture-and-async.md)

## Modules & Runtime
- [JPMS Module System](modules-and-runtime/jpms-module-system.md)
- [Reflection & Method Handles](modules-and-runtime/reflection-and-method-handles.md) — `invokedynamic`
- [VarHandle & Unsafe](modules-and-runtime/varhandle-and-unsafe.md) — low-level access, the JMM
- [jlink & Runtime Images](modules-and-runtime/jlink-and-runtime-images.md) — AppCDS, native-image

## I/O
- [I/O Streams & Readers](io/io-streams-and-readers.md) — `java.io`, the decorator pattern
- [NIO & Channels](io/nio-and-channels.md) — buffers, selectors, the reactor model
- [The Stream API](io/stream-api.md) — functional data pipelines (distinct from I/O streams)

## Self-Test
- [Review Questions](review-questions.md)
