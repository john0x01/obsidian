# Threads and the OS

A Java *platform thread* is a thin wrapper over a single operating-system thread — the JVM's `java.lang.Thread` maps **1:1** onto a kernel-scheduled thread of execution. This is the foundation everything else in `java.util.concurrent` is built on, and understanding its cost is what motivates virtual threads.

## The 1:1 Mapping and Its Cost

When you call `new Thread(...).start()`, the JVM asks the OS to create a native thread. The OS owns its scheduling and gives it a contiguous stack — typically **hundreds of KB to ~1MB** (default `-Xss` is around 512KB–1MB depending on platform). That memory is reserved up front per thread, so a few thousand threads consume gigabytes of address space and real memory for hot stacks. Two costs dominate:

- **Memory**: large fixed stacks. This is the hard ceiling — you run out of RAM long before the CPU is busy.
- **Scheduling**: the OS kernel time-slices threads. A context switch saves/restores registers, swaps stacks, and pollutes CPU caches and the TLB. Switching among thousands of mostly-idle threads is pure overhead.

Unlike JavaScript's single-threaded event loop, Java gives you *true* parallelism — N platform threads can run on N cores simultaneously. The price is that threads are a scarce, expensive resource you must pool, not spawn freely.

## Lifecycle and States

`Thread.getState()` returns one of six values (`Thread.State`):

```
NEW ──start()──► RUNNABLE ──┬─► BLOCKED        (waiting for a monitor lock)
                            ├─► WAITING        (wait/join/park, no timeout)
                            └─► TIMED_WAITING  (sleep/wait(t)/join(t))
                                   │
RUNNABLE ──run() returns/throws──► TERMINATED
```

A subtlety: `RUNNABLE` covers both "actually executing on a core" and "ready but waiting for a CPU" *and* "blocked in a syscall on I/O" — the JVM cannot distinguish kernel I/O waits from running, so a thread blocked reading a socket still shows `RUNNABLE`. `BLOCKED` specifically means contention on a `synchronized` monitor.

## start vs run, Daemon vs User

`start()` requests a new OS thread that then invokes `run()`. Calling `run()` directly is a classic beginner trap: it just executes the body **on the current thread**, with no concurrency. You can only `start()` a given `Thread` once — a second call throws `IllegalThreadStateException`.

Threads are either **user** or **daemon** (`setDaemon(true)` before `start()`). The JVM stays alive as long as any *user* thread runs; daemon threads (GC helpers, background pollers) are abandoned abruptly at JVM exit and never block shutdown. Pitfall: daemon threads get no chance to run `finally` blocks or flush buffers on shutdown — don't hold critical state in them.

## Cooperative Interruption

Java has **no safe preemptive kill**. `Thread.stop()` is deprecated for removal because it could unlock monitors mid-mutation, leaving shared state corrupted. Instead, interruption is *cooperative*:

- `t.interrupt()` sets the thread's **interrupt flag**.
- Methods that block (`sleep`, `wait`, `join`, blocking NIO) check it and throw `InterruptedException`, which **clears** the flag.
- CPU-bound loops must poll `Thread.interrupted()` (clears) or `isInterrupted()` (doesn't) themselves.

The senior discipline: never swallow `InterruptedException`. Either propagate it, or restore the flag with `Thread.currentThread().interrupt()` so callers up the stack can still observe cancellation. Swallowing it silently breaks the entire cancellation chain.

## Why Raw Threads Don't Scale — the C10k Framing

The classic *C10k problem* asks: how do you serve 10,000+ concurrent connections? With thread-per-request on platform threads you can't — 10k threads × ~1MB stacks is ~10GB, plus crippling scheduling overhead, even though each connection is **mostly idle waiting on I/O**. The industry's two historical answers were both unsatisfying:

1. **Thread pools**: cap threads, queue work. But a blocking call (DB query, HTTP fetch) holds a pooled thread hostage while idle, so the pool's size caps your real concurrency. Under-provision and you stall; over-provision and you pay the memory/switching tax.
2. **Async / non-blocking** (Netty, reactive, like JS): never block a thread; use callbacks/futures and an event loop. This scales beautifully but inflicts *callback hell* / "colored functions" and loses the readable, debuggable straight-line `try/catch`/stack-trace style.

This tension — simple-but-unscalable vs scalable-but-complex — is exactly what **virtual threads** dissolve: keep the simple blocking thread-per-request code, but make each thread cost a few KB so millions are feasible. Platform threads remain the carriers underneath; they never go away, they just stop being the unit of concurrency.

## See Also

- [[virtual-threads]]
- [[executors-and-thread-pools]]
- [[exceptions-and-error-handling]]
- [[03-computer-systems/operating-systems/threads|OS Threads]]
- [[03-computer-systems/concurrency-and-parallelism/threads-and-processes|Threads and Processes]]
- [[03-computer-systems/operating-systems/scheduling|Scheduling]]
