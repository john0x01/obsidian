# Worker Threads

`worker_threads` gives Node a way to run JavaScript on **real OS threads**, each with its own V8 *Isolate*, its own libuv event loop, and its own heap. The point is not concurrency for I/O (the single-threaded event loop already excels at that) but **parallelism for CPU-bound work** — hashing, compression, image transforms, parsing huge payloads — that would otherwise block the main loop and starve every other request.

## The Isolate Model

The mental model that matters: a Worker is *not* a thread that shares your variables. V8 is built around the *Isolate* — a fully self-contained instance of the engine with its own heap, its own garbage collector, and its own microtask queue. A `new Worker()` spins up a new Isolate on a new thread. Two Isolates cannot see each other's objects at all; there is no shared address space for ordinary JS values. This is deliberate: it sidesteps the entire class of data-race bugs that plague shared-memory threading in C++/Java. You get threads without locks because, by default, you get *isolation* without sharing.

```
Process
├─ Main thread:   Isolate A | heap A | event loop A
├─ Worker 1:      Isolate B | heap B | event loop B   (real OS thread)
└─ Worker 2:      Isolate C | heap C | event loop C   (real OS thread)
```

Each Worker boots its own runtime: it re-parses and re-compiles the entry module, re-initializes globals, and has its own `require`/`import` cache. Threads are cheaper than processes (shared process memory for the binary, no `fork()`/`exec()`) but a Worker still costs on the order of single-digit-MB of baseline V8 heap plus startup latency to compile its code. They are not free; do not spawn one per request.

## Messaging and the Copy Boundary

Communication is message-passing over a port. `parentPort.postMessage(value)` / `worker.on('message', ...)` move data between Isolates using the **structured clone algorithm** — the same algorithm used by browser `postMessage`. The value is *deep-copied* into the receiving heap. Functions, prototypes, class identity, and closures do not survive; you get plain data back (Maps, Sets, Dates, typed arrays, and circular references *do* survive). The cost is proportional to the serialized size, and it is synchronous on the sending thread — cloning a 200 MB object to ship it to a Worker can itself block the loop you were trying to protect.

Two escape hatches avoid the copy:

- **Transferables.** Pass a second argument listing objects to *transfer* ownership rather than clone: `postMessage(buf, [buf])`. The underlying `ArrayBuffer` is detached on the sender (length becomes 0) and re-attached on the receiver — a pointer hand-off, O(1) regardless of size. `MessagePort` instances are also transferable.
- **`SharedArrayBuffer`.** This is the one kind of memory genuinely *shared* across Isolates. Both threads map the same backing store; writes by one are visible to the other. There is no clone and no transfer — you `postMessage` a reference and both sides hold live views. This reintroduces real data races, so coordination must go through `Atomics` (`Atomics.add`, `Atomics.wait`, `Atomics.notify`) which provide atomic RMW operations and a futex-like blocking primitive. `Atomics.wait` is the *only* way to block a JS thread synchronously and is legal **only** inside a Worker, never on the main thread.

`MessageChannel` gives you a standalone `{ port1, port2 }` pair you can transfer to set up direct Worker-to-Worker links, bypassing the parent as a relay — useful for pipelines and mesh topologies.

## Edge Cases and Pitfalls

- **Workers are not a thread pool.** Each `new Worker` is a fresh runtime. For request-style workloads you want a *pool* of long-lived workers (`piscina` is the canonical library; rolling your own means recycling workers and queuing tasks). Re-spawning per task pays the compile/startup cost every time and will be slower than the synchronous version you were trying to escape.
- **libuv's threadpool is a separate thing.** `crypto.pbkdf2`, `fs` calls, and DNS already run on libuv's internal pool (default size 4, set via `UV_THREADPOOL_SIZE`). That offloads native work but *not* your JS. Worker Threads exist precisely because pure-JS CPU work has nowhere else to go.
- **Shared-nothing has surprising reach.** Native addons may hold process-global state; some modules assume single-threaded init. Workers also share file descriptors and the process-wide `stdout`/`stderr`, which can interleave.
- **Termination is abrupt.** `worker.terminate()` kills the Isolate immediately — no `finally`, no flush. Design for graceful shutdown via a message.
- **`Atomics.wait` deadlocks** if the notifying thread never runs; treat shared-memory coordination with the same rigor as C threads.

## Philosophy

Node's bet is that **isolation is the default, sharing is opt-in**. Most parallelism needs are embarrassingly parallel batch work, well served by copying inputs out and results back. Shared memory exists for the rare hot path where copy cost dominates — and there you pay with all the complexity of classical concurrency. Use Workers when profiling shows a synchronous CPU hot spot blocking the loop; reach for processes when you need fault isolation or a separate memory limit; and resist the urge to thread I/O — the event loop already does that better.

## See Also

- [[cluster-and-child-processes]]
- [[async-context]]
- [[01-programming-foundations/languages/javascript/concurrency/web-workers|Web Workers]]
- [[01-programming-foundations/languages/javascript/concurrency/shared-memory-and-atomics|Shared Memory and Atomics]]
- [[03-computer-systems/concurrency-and-parallelism/threads-and-processes|Threads and Processes]]
- [[07-performance-engineering/cpu-bound-optimization|CPU-Bound Optimization]]
