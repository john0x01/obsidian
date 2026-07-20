# Virtual Threads

A *virtual thread* (Project Loom, finalized in Java 21 via JEP 444) is a `java.lang.Thread` that the JDK — not the OS — schedules onto a small pool of platform *carrier* threads. It solves the problem that motivated [[threads-and-the-os|platform threads]]' scarcity: it makes a thread cost a few hundred bytes of heap instead of a ~1MB kernel stack, so you can run *millions* concurrently and go back to writing straight-line, blocking, thread-per-request code that scales.

## The Mechanism: Continuations and Carriers

A virtual thread is a thin object wrapping a **delimited continuation** plus a scheduler. Its call stack lives on the **heap** as stack-chunk objects that grow and shrink on demand — there is no reserved fixed stack. To actually execute, the virtual thread must be **mounted** onto a carrier (a platform thread from a JDK-managed `ForkJoinPool` running in FIFO mode, sized by default to `Runtime.availableProcessors()`; tunable via `jdk.virtualThreadScheduler.parallelism`).

The pivotal move is what happens on a blocking call:

```
VT running on carrier C1
   │  socket.read()  ── blocks ──►  JDK intercepts
   ▼
UNMOUNT: copy VT stack to heap, C1 is freed
   │            (C1 now runs some OTHER virtual thread)
   ▼  I/O completes
MOUNT: VT resumes on some carrier (maybe C7), stack restored
```

When a virtual thread blocks on a JDK-native blocking operation — NIO socket I/O, `Thread.sleep`, `BlockingQueue`, `ReentrantLock`, `Future.get` — the runtime **unmounts** it: its continuation yields, the stack is parked on the heap, and the carrier is immediately handed to another runnable virtual thread. When the I/O completes, the virtual thread is **remounted** onto any free carrier and resumes exactly where it left off. The carrier's OS stack is used *only while running*; idle waiting costs nothing but heap. The JDK was extensively retrofitted so that the whole `java.net`/`java.nio`/`java.util.concurrent` surface unmounts instead of blocking the carrier.

## The Mental Model vs the JS Event Loop

This is the sharp contrast the JS/Node engineer should internalize. Node achieves massive I/O concurrency with **one** thread and an [[03-computer-systems/concurrency-and-parallelism/event-loops|event loop]]: your code yields at every `await`, and the loop multiplexes callbacks. The cost is *function coloring* — async infects call chains — and a single CPU-bound function stalls **every** connection.

Virtual threads give you the same I/O scalability with the *opposite* ergonomics:

- **No coloring.** You write `var user = db.query(...)` — blocking, sequential, `try/catch` works, and the stack trace is the real logical call chain. The unmount is the moral equivalent of an implicit `await`, but the compiler and you never see it.
- **Real parallelism.** N carriers run on N cores; unlike the single JS thread, CPU work on one virtual thread does not freeze the others.
- **Graceful degradation.** A misbehaving CPU-bound virtual thread ties up *one* carrier, not the whole world. In Node the same mistake halts the entire process.

Think of the carrier pool as a hidden event loop that the JDK drives for you, with continuations standing in for the state machines that `async/await` compiles to elsewhere.

## Pinning: The One Leak in the Abstraction

Unmounting requires the JVM to capture the continuation, which it cannot do across certain frames. A **pinned** virtual thread blocks while still occupying its carrier, defeating the model. Two causes historically mattered:

1. **`synchronized`** — through Java 21–23, blocking inside a `synchronized` block or method pinned the carrier. The fix was to swap `synchronized` for `ReentrantLock` around I/O-bearing critical sections. **Java 24 (JEP 491) reworked the JVM so `synchronized` no longer pins in the common case** — a major relief, and the state you should assume on Java 25. Still, treat legacy libraries' `synchronized` I/O with suspicion if you run older JDKs.
2. **Native frames** — a `native`/JNI frame or certain foreign-function calls on the stack still pin, because the JVM cannot relocate a C stack. This has no general fix; it is inherent.

Diagnose pinning with the JFR event `jdk.VirtualThreadPinned`. A pool of, say, 8 carriers fully pinned is a hard concurrency ceiling that reintroduces the very problem virtual threads exist to remove.

## Senior Pitfalls and Philosophy

- **Never pool virtual threads.** Pooling exists to amortize a costly resource; virtual threads *are* the cheap resource. Use `Executors.newVirtualThreadPerTaskExecutor()` or `Thread.startVirtualThread(...)` — one per task, unbounded.
- **They are for I/O, not CPU.** A billion-iteration loop gains nothing from a virtual thread; it just occupies a carrier. Bound genuine parallelism with platform threads or a fixed pool.
- **Bound resources, not threads.** With unlimited threads, your database connection pool or a `Semaphore` becomes the real limiter — size those deliberately.
- **`ThreadLocal` becomes a hazard.** Millions of threads each holding thread-local state is a memory blowup; this is precisely what [[structured-concurrency|scoped values]] were designed to replace.

The philosophy: Loom re-legitimizes *blocking* as the default. For twenty years "don't block a thread" was gospel because threads were precious. Loom makes the thread cheap, so the thread-per-request model — readable, debuggable, honest about causality — wins again, and reactive plumbing becomes a specialized tool rather than the default.

## See Also

- [[threads-and-the-os]]
- [[structured-concurrency]]
- [[completablefuture-and-async]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime & Event Loop]]
- [[10-frameworks-and-stacks/node-js/architecture/libuv-and-event-loop|Node Event Loop]]
