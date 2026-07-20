# Escape Analysis and Optimizations

Once HotSpot's C2 (or Graal) compiler has a hot method and its profile, it applies a cascade of optimizations that make idiomatic, allocation-heavy Java run like hand-tuned code. The keystone is **escape analysis** — proving where an object's lifetime is confined — which unlocks eliminating allocations and locks entirely. Understanding these transforms is what separates "the JIT is magic" from knowing why a benchmark lies.

## Escape Analysis

The compiler classifies each allocation by how far its reference *escapes* the creating method:

- **NoEscape** — never visible outside the method (a local `Point` used and dropped).
- **ArgEscape** — passed to a callee but not stored globally.
- **GlobalEscape** — stored in a field, returned, or handed to another thread.

NoEscape objects need never exist as heap objects.

## Scalar Replacement (Not Stack Allocation)

The single most misunderstood JIT optimization: HotSpot does **not** allocate whole objects on the stack. Instead, for a NoEscape object it performs **scalar replacement** — the object is dissolved and its fields are promoted to individual local variables that live in registers or stack slots. The `new` vanishes; there is no header, no GC pressure, no dereference. A `new Point(x,y)` that never escapes becomes two `int` locals.

```text
before:  p = new Point(a,b); use(p.x + p.y)
after:   px = a; py = b; use(px + py)   // no object exists
```

This is why allocation "cost" in Java is often near zero for short-lived objects — but only *after* C2 compiles the method, and only if escape analysis succeeds. One reference stored in a field or returned flips the object to GlobalEscape and the allocation reappears on the heap (see [[heap-and-allocation]]). Graal's *partial* escape analysis goes further, keeping the object scalarized on paths where it doesn't escape and materializing it only on the paths where it does.

## Lock Elision and Coarsening

Escape analysis also proves a lock object is thread-confined. **Lock elision** then removes the `monitorenter`/`monitorexit` entirely — the classic case being a `StringBuffer` or a `synchronized` collection used purely locally. **Lock coarsening** merges repeated lock/unlock on the *same* monitor across adjacent blocks into one larger critical section, trading a hair of latency to avoid ping-ponging the lock. Both explain why "just synchronize it" often costs nothing measurable in single-threaded hot paths.

## Inlining: The Enabling Optimization

Inlining is called the mother of all optimizations because it doesn't just remove call overhead — it *merges* caller and callee into one scope where every other optimization (escape analysis, constant folding, dead-code elimination) can then see across the old boundary. HotSpot inlines by size and hotness (small methods almost always; hot ones up to a larger budget, bounded by an inline-depth limit). Virtual calls are inlined speculatively: if profiling shows a call site is monomorphic (one receiver type) or bimorphic, C2 inlines the likely target behind a guard and deoptimizes if a surprise type shows up. **Megamorphic** sites (many types, e.g. an over-abstracted interface hierarchy) defeat inlining and stay slow — a real design cost of excessive polymorphism.

## Intrinsics and Speculation

Some methods are too important to leave to general compilation. **Intrinsics** are hand-written, CPU-optimal machine sequences the JIT substitutes by *recognizing the method*, not compiling its bytecode: `Math.sqrt`, `System.arraycopy`, `Integer.bitCount`, `String.indexOf`, array-fill, CRC, and vector ops map to dedicated instructions. Beyond intrinsics, all of C2's aggressive moves are **speculative** — guarded by profile-derived assumptions that trigger deoptimization when violated (see [[execution-engine-and-jit]]). The compiler bets, and the guard is the insurance.

## The Microbenchmark Trap

These optimizations make naïve benchmarks meaningless. The JIT will:

- **Dead-code eliminate** a computation whose result is unused — your loop measures nothing.
- **Constant-fold** inputs the compiler can see are constants, precomputing the "work."
- **Scalar-replace** allocations that a real caller (where the object escapes) would keep — under-reporting real GC cost.
- **Loop-unroll or hoist** invariants out of the timed region.
- Measure the interpreter/C1 if warmup never reaches C2.

**JMH** exists to defeat exactly this: forked JVMs isolate profile pollution, dedicated warmup iterations reach steady-state C2, `Blackhole.consume(...)` prevents dead-code elimination, and `@State`-scoped inputs block constant folding. The mental model: you are not benchmarking an expression, you are benchmarking whatever the JIT decided to leave after optimizing it. See [[07-performance-engineering/benchmarking|Benchmarking]].

## Senior Pitfalls

- "Objects are stack-allocated" — no; they're scalar-*replaced*, and only when NoEscape. Returning or storing the object silently reintroduces heap allocation.
- Trusting a hand-rolled `System.nanoTime()` micro-loop over JMH.
- Fighting the GC by object-pooling short-lived objects the JIT would have eliminated anyway (see [[gc-fundamentals]]) — pooling can *worsen* things by forcing GlobalEscape.

## See Also

- [[execution-engine-and-jit]]
- [[bytecode-and-class-files]]
- [[heap-and-allocation]]
- [[gc-fundamentals]]
- [[07-performance-engineering/benchmarking|Benchmarking]]
