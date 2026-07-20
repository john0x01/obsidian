# Execution Engine and JIT

HotSpot runs your program twice over: it *interprets* bytecode immediately for fast startup, then *profiles* which code is hot and compiles it to native machine code on the fly. This is the JVM's answer to the interpreter-vs-compiler dilemma — pay nothing up front, then spend compilation effort only where it repays itself, using runtime information an ahead-of-time compiler never has.

## Interpreter Plus Tiered Compilation

HotSpot ships a **template interpreter**: at startup it generates small machine-code stubs for each bytecode, so even "interpreted" execution is native dispatch, not a switch loop. Alongside it sit two JIT compilers:

- **C1 (client)** — fast to compile, modest optimization, inserts profiling counters.
- **C2 (server)** — slow, aggressive, profile-driven; produces the fastest code.

**Tiered compilation** (the default) stitches these into five levels: Tier 0 interpreter, Tiers 1–3 are C1 variants (trivial, limited-profile, full-profile), Tier 4 is C2. The usual journey is 0 → 3 → 4: run interpreted, get promoted to profiling C1 to gather data cheaply, then recompiled by C2 once truly hot. If the C2 queue backs up, methods sit at Tier 3; trivial methods stop at Tier 1 since C2 buys nothing. The point of this dance is that C2's expensive optimization is fed by *real* profile data (branch frequencies, receiver types) collected during the C1 phase.

## Thresholds and Profiling

Each method carries an **invocation counter** and per-loop **back-edge counters**. When their sum crosses a tier threshold, the method is queued for compilation (compilation itself runs on background threads, so hot code keeps executing meanwhile). Under tiered mode the promotion knobs are roughly `Tier3InvocationThreshold` (~200) and `Tier4InvocationThreshold` (~5000) — treat these as ballpark, since HotSpot adjusts them adaptively; the legacy non-tiered `CompileThreshold` of 10000 no longer governs the default path. Profiling records more than counts: branch taken/not-taken ratios, observed receiver classes at call sites, null-ness, and type checks — the raw material for speculation.

## On-Stack Replacement

A method with a long-running loop may never *return* to trigger normal recompilation. **OSR** solves this: when a back-edge counter trips, the JIT compiles a special entry point and swaps the running interpreted frame for a compiled one *mid-method*, transferring live locals into the optimized frame. This is why a program can visibly speed up inside a single hot loop without the enclosing method ever being called again. OSR code is specialized to that loop and is often slightly less optimal than a normal compilation.

## Deoptimization and Uncommon Traps

C2 optimizes *speculatively*: using Class Hierarchy Analysis it may inline a virtual call as if monomorphic, or prune a branch profiling never took. These bets are protected by cheap **guards**. When a bet fails — a new subclass is loaded, the cold branch finally executes, a null appears — control hits an **uncommon trap**: the compiled frame is *deoptimized*, rebuilt as interpreter frames (the reverse of OSR), and execution continues correctly while the stale native code is discarded and eventually recompiled without the broken assumption. This ability to *undo* optimization is precisely what lets HotSpot gamble more aggressively than a static compiler dares.

## The Code Cache

Compiled native code lives in the **code cache**, a fixed off-heap region (default reserved ~240 MB under tiered), segmented since Java 9 into non-method, profiled, and non-profiled areas so short-lived C1 code doesn't fragment long-lived C2 code. If it fills, the JIT *shuts off* — you fall back to the interpreter and see "CodeCache is full. Compiler has been disabled." Large, heavily-modular apps (or ones generating classes at runtime) can hit this and mysteriously slow down late in their run.

## GraalVM and the Broader Picture

The Graal compiler — written in Java and plugged in through JVMCI (the JVM Compiler Interface) — can replace C2 as the top tier, bringing more aggressive inlining and *partial* escape analysis. It is also the engine behind GraalVM Native Image's AOT path. This makes the JIT-vs-AOT choice concrete: JIT wins peak throughput via runtime profiles and speculation; AOT wins startup and memory by compiling ahead, forfeiting deopt. See [[07-performance-engineering/jit-vs-aot|JIT vs AOT]] and, for the same interpret-then-compile shape in JavaScript, [[01-programming-foundations/languages/javascript/engine-internals/jit-compilation|JS JIT Compilation]] (Ignition → TurboFan mirrors interpreter → C2, deopt included).

## Senior Pitfalls

- Measuring cold code: the first thousands of iterations run interpreted or in C1. Benchmarks without warmup measure the wrong thing (see [[escape-analysis-and-optimizations]]).
- Blaming the GC for a latency cliff that is actually deopt storms or a full code cache.
- Assuming `-Xint` or disabling C2 is "safer" — you forfeit the entire performance model the runtime is built around.

## See Also

- [[jvm-architecture]]
- [[bytecode-and-class-files]]
- [[escape-analysis-and-optimizations]]
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]
- [[01-programming-foundations/languages/javascript/engine-internals/jit-compilation|JS JIT Compilation]]
