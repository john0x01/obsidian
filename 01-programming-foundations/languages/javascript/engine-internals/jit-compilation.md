# JIT Compilation

A JavaScript engine can't compile ahead of time — the code arrives at runtime and its types are unknown until it runs — yet interpreting bytecode forever is too slow for hot loops. V8's answer is **tiered, profile-guided JIT compilation**: run code in a cheap tier first, measure what's hot and what types it sees, then recompile the hot parts into increasingly specialized machine code, while keeping a way to fall back when assumptions break. Each tier trades compile cost and memory against execution speed.

## The Tiers

```
Ignition (interpreter)        cheap to produce, smallest memory, slowest exec
   |  hot?                     (collects type feedback)
   v
Sparkplug (baseline JIT)      no optimization, fast to compile, ~1.5x faster
   |  hotter?
   v
Maglev (mid-tier optimizer)   light speculation, fast compile, good speed
   |  hottest?
   v
TurboFan (top-tier optimizer) heavy optimization, slow compile, fastest exec
```

- **Ignition** interprets bytecode and records **type feedback**. It is the baseline of correctness and the profiler. (See [[parsing-and-bytecode]].)
- **Sparkplug** is a **non-optimizing baseline compiler**: it translates bytecode to machine code almost mechanically (a single near-linear pass, no IR, no optimization), so it compiles extremely fast and just removes interpreter dispatch overhead. Its frames are compatible with the interpreter's, which makes switching tiers cheap.
- **Maglev** is a **mid-tier optimizing compiler**: it builds a simpler SSA-style IR and does *some* speculative optimization (inline caches, type specialization) but compiles far faster than TurboFan — bridging the large gap between baseline and top tier so functions get meaningful speedups sooner.
- **TurboFan** is the **top-tier optimizer**: a full "sea of nodes" IR with aggressive inlining, escape analysis (scalar replacement), redundancy elimination, and range/type narrowing. Expensive to run, so reserved for the hottest code.

Promotion is driven by counters/heuristics (invocation counts, loop iteration counts, "interrupt budget"). Compilation of the optimizing tiers happens on **background threads**, so it doesn't block the main thread — the function keeps running in its current tier until the optimized code is ready, then is swapped in.

## Speculative Optimization from Type Feedback

The reason a dynamically-typed language can be JIT-compiled to fast code at all is **speculation**. JavaScript has no static types, but a given call site usually sees *the same* types over and over (the monomorphism assumption — see [[hidden-classes-and-inline-caches]]). The optimizer reads the feedback vector and *assumes* those observed types and object shapes hold, then emits code specialized for them: a property load becomes a fixed offset, `a + b` becomes an integer add, a polymorphic call becomes a direct (inlined) call.

To stay correct, every speculation is **guarded**: the optimized code begins with cheap checks ("is `obj` still this hidden class? is `x` still a Smi?"). If a guard fails, the code **deoptimizes** — bails out to Ignition at the equivalent bytecode point and continues correctly. Optimized code is therefore an *unsound but checked* fast path layered over a sound interpreter. (See [[deoptimization]].)

```js
function area(s) { return s.w * s.h; }
// after feedback: s is always {w,h} Smis with one hidden class
// TurboFan emits:  check shape(s)==Shape_WH; check Smi(w),Smi(h); imul
// if a caller later passes {w,h,z} or a float -> guard fails -> deopt
```

## On-Stack Replacement (OSR)

A function with a long-running loop (think a hot `for` over a big array) would never benefit from optimization if V8 had to wait for the *next call* to install optimized code — the current call might run for seconds. **On-stack replacement** solves this: V8 compiles an optimized version of the function *while it is mid-execution*, then transfers the live state (registers, locals, loop counter) from the running frame into the optimized frame and resumes inside the loop. OSR is how a single hot loop gets fast without restarting. The reverse transition — deopt — is the same machinery in the other direction, reconstructing interpreter frame state from optimized state.

## The Speed / Memory / Latency Trade-offs

Each tier is a point on a curve:

- **Higher tiers = faster steady-state, but** more compile time (CPU and battery), more memory (machine code is bigger than bytecode), and a latency cost if a function optimizes then deopts repeatedly.
- **Cold code should stay cheap.** Most functions never leave Ignition/Sparkplug, and that's correct — optimizing run-once code is pure waste. This is why benchmarking the first iteration is meaningless; you measure the interpreter, not the steady state.
- **Tier-up takes time.** A loop needs to iterate enough to trigger OSR; a function needs enough calls. Short, bursty workloads (serverless cold starts, one-shot scripts) may never reach TurboFan at all — a reason `--max-opt`/lower-tier-only modes and startup snapshots matter for them.

**Senior implications**: write code that *stays optimizable* — consistent argument types, stable object shapes, avoid features that historically defeated optimization or forced lower tiers. Don't hand-inline what TurboFan inlines for free, but do avoid megamorphic call sites and shape churn that prevent it. And treat "fast after warm-up" as the design point: keep hot paths monomorphic, keep cold paths from dragging the tiering system into pointless work.

## See Also
- [[parsing-and-bytecode]] — the bytecode all tiers compile from
- [[hidden-classes-and-inline-caches]] — the type feedback that drives speculation
- [[deoptimization]] — what happens when speculation fails
- [[v8-architecture]] — the full pipeline these tiers sit in
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]] — JIT vs ahead-of-time trade-offs
- [[07-performance-engineering/compilation-optimization|Compilation Optimization]] — optimization passes in general
