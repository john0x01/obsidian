# Safepoints

A **safepoint** is a point in a thread's execution where all of its references (the roots the GC and runtime care about) are in a known, consistent state, so the JVM can safely inspect or rewrite the world. Operations like a stop-the-world GC pause, deoptimization, class redefinition, or a heap dump can only run when the threads they touch are parked at a safepoint. Understanding safepoints explains *why* "the JVM paused" and why some profilers lie.

## Why They Exist

The VM periodically needs a globally consistent view of the heap and stacks: move objects (evacuating GC), patch compiled code (deopt), or walk every stack precisely (GC root scanning). Doing that while a thread is mid-instruction — a half-updated object header, a reference live only in a register — would corrupt state. A safepoint is the agreed contract: *"at these points, my stack is walkable and my references are settled."* The JIT emits **oopmaps** describing where references live at each safepoint so the GC can find and update them precisely.

## Reaching a Safepoint

The VM cannot force a thread to stop mid-instruction; threads must **poll** cooperatively:

- **JIT-compiled code** executes a cheap *safepoint poll* — historically a read of a special guard page — at method returns and loop back-edges. When a safepoint is requested, the VM protects that page; the next poll faults, traps into the runtime, and the thread parks. (Modern HotSpot uses thread-local handshake polls rather than a single global page.)
- **Interpreted code** checks a per-thread flag between bytecodes.
- **Blocked / JNI threads** are already "safe" (in native code with no Java frames to move); they're marked and re-checked when they return.

A **global safepoint** requests every application thread to park; the VM then runs the sensitive operation and releases them. **Thread-local handshakes** let the VM pause and inspect *one* thread without a global pause — used for lighter operations (e.g. stack sampling, deopt of a single thread).

## Time to Safepoint (TTSP)

The reported GC "pause" has two parts: **time-to-safepoint** (from the request until the *last* thread parks) plus the operation itself. TTSP is bounded by the slowest thread to reach a poll — so a single thread stuck in a poll-free region stalls *everyone*:

- **Counted loops** (`for (int i=0; i<n; i++)`) historically had their back-edge polls elided for speed, so a long counted loop could run for many milliseconds before polling. `-XX:+UseCountedLoopSafepoints` reinstates them.
- Large **array copies**, `fill`, or intrinsics, and long **JNI** calls, are effectively poll-free.

`-Xlog:safepoint` (and `safepoint+stats`) exposes TTSP and per-phase timing — the first thing to check when pauses are larger than the collector should cause.

## Safepoint Bias in Profilers

Many sampling profilers take stacks via JVMTI `GetStackTrace`/`GetAllStackTraces`, which **only sample at a safepoint** — so a request pauses the thread at the *next poll*, not where it actually was. Time gets attributed to safepoint locations (method returns, loop edges), not the true hot instruction, and poll-free hot regions become invisible. This is **safepoint bias**. Tools built on `AsyncGetCallTrace` / perf (e.g. **async-profiler**) sample at arbitrary points via signals and avoid it — which is why their flame graphs often disagree with a safepoint-biased profiler.

## Senior Pitfalls & Mental Model

- A "GC pause" you can't explain by heap work is often **TTSP** — hunt the straggler thread, not the collector.
- Concurrent collectors (ZGC, Shenandoah) still take *brief* safepoints for root scanning and phase switches; "pauseless" means the heavy work is concurrent, not that safepoints vanish.
- Don't trust a safepoint-biased profiler for micro-hotspots; cross-check with async-profiler.
- Mental hook: **the JVM can only change the world when everyone it needs is standing still** — safepoints are how it gets everyone to stand still, and TTSP is how long the stragglers make everyone wait.

## See Also

- [[gc-fundamentals]]
- [[g1-collector]]
- [[low-pause-collectors]]
- [[execution-engine-and-jit]]
- [[threads-and-the-os]]
- [[07-performance-engineering/profiling|Profiling]]
