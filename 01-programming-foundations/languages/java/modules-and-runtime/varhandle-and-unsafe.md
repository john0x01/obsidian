# VarHandle And The Retirement Of Unsafe

A `VarHandle` is a typed, low-level reference to a variable — an object field, an array element, or a region of off-heap memory — that exposes a menu of *access modes* whose memory-ordering guarantees map directly onto the Java Memory Model. It is the supported, safe replacement for the field-and-memory tricks libraries used to reach for in `sun.misc.Unsafe`, giving lock-free code the same primitives without the footguns.

## VarHandle And Access Modes

You obtain a `VarHandle` from `MethodHandles.lookup().findVarHandle(Type.class, "field", FieldType.class)`, or `MethodHandles.arrayElementVarHandle(...)`, or over a `ByteBuffer`/`MemorySegment`. Its power is that a single handle offers the *same* variable under different ordering contracts, so you pay only for the ordering you need. Mapped onto the JMM (see [[java-memory-model]], with the fence-level detail in [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]):

- **Plain** (`get`/`set`) — ordinary field access; no atomicity or ordering guarantee beyond the access itself, freely reorderable. Same cost as a normal read/write.
- **Opaque** (`getOpaque`/`setOpaque`) — bitwise atomic and *coherent* (per-variable order is respected and progress is guaranteed — no hoisting a read out of a spin loop), but establishes no ordering with respect to *other* variables.
- **Acquire/Release** (`getAcquire`/`setRelease`) — a release store publishes all prior writes to a thread that later performs an acquire load of the same variable: a one-directional happens-before edge, cheaper than full volatile because it needs only a one-sided fence.
- **Volatile** (`getVolatile`/`setVolatile`) — full `volatile` semantics: sequentially consistent among volatile-mode accesses (see [[volatile-and-synchronized]]).

On top sit the atomic updaters — `compareAndSet`, `compareAndExchange` (with acquire/release variants), `getAndAdd`, `getAndSet`, and `weakCompareAndSet` (which may fail spuriously but compiles to a bare hardware CAS ideal for retry loops). This is the toolkit for [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]]; the atomics themselves are [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]].

The design point is *graduated cost*: before VarHandles you had `volatile` (a full barrier) or nothing. Acquire/release and opaque expose the middle rungs of the hardware's ordering ladder that experts previously reached only through `Unsafe`.

## What Unsafe Did

`sun.misc.Unsafe` was an internal, unsupported class that nonetheless became load-bearing across the ecosystem (Netty, off-heap caches, the LMAX Disruptor, many serializers). It offered:

- Off-heap allocation — `allocateMemory`/`freeMemory` and raw `getLong`/`putLong` at an address.
- Direct field access by offset — `objectFieldOffset` plus `getInt`/`putInt`, and CAS (`compareAndSwapInt`).
- Memory fences — `loadFence`/`storeFence`/`fullFence`.
- Constructor-bypassing allocation (`allocateInstance`), throwing undeclared exceptions, and `park`/`unpark`.

It was powerful and completely unsafe: raw pointers, no bounds checks, JVM crashes on a bad offset, and no encapsulation whatsoever.

## The Deprecation Path And FFM

The JDK is retiring `Unsafe` by splitting its responsibilities across two supported APIs:

- **On-heap** field and array CAS/ordering → **VarHandle** (JEP 193, Java 9).
- **Off-heap** memory and native interop → the **Foreign Function & Memory API** (`java.lang.foreign`), finalized as JEP 454 in JDK 22 (Project Panama).

The FFM API allocates and accesses native memory through `MemorySegment`, `MemoryLayout`, and — critically — `Arena`, which gives memory *bounded, deterministic* lifetime plus spatial and temporal safety: a confined arena enforces thread confinement, a shared arena is safely closeable, and access after close throws instead of corrupting the heap. `Linker` additionally performs native downcalls, replacing hand-written JNI.

Accordingly, the **memory-access methods of `sun.misc.Unsafe` were deprecated for removal** (JEP 471, JDK 23), with removal slated for a future release — the exact version is not yet fixed, so treat any specific number as tentative. The migration is the platform practicing what it preaches: the same strong-encapsulation logic that closed `sun.*` in [[jpms-module-system]] now closes the last widely-used backdoor, but only after shipping replacements that are *safer and no slower* — a `VarHandle` rides the JIT like a method handle (see [[reflection-and-method-handles]]), and FFM's `MemorySegment` bounds checks are elided by the optimizer in hot loops. The senior takeaway: reach for VarHandle for concurrent field access and FFM for off-heap, and audit dependencies still pinned to `Unsafe`.

## See Also
- [[java-memory-model]]
- [[volatile-and-synchronized]]
- [[reflection-and-method-handles]]
- [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]
- [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]]
- [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]]
- [[locks-and-atomics]] — higher-level atomics built on this
