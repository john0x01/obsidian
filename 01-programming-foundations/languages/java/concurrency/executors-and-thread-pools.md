# Executors and Thread Pools

The `Executor` framework decouples *what* runs (a task) from *how, when, and on which thread* it runs. Because platform threads are expensive to create and destroy (see [[threads-and-the-os]]), the answer is almost always a **thread pool**: a fixed set of worker threads pulling tasks from a queue. This is the classic pre-Loom concurrency model, and knowing its internals is what separates safe pool configuration from cargo-culting `Executors` factory methods.

## The Three-Layer Interface Hierarchy

The API is deliberately layered. `Executor` is a single method — `execute(Runnable)` — the minimal contract for "run this somehow." `ExecutorService` adds *lifecycle and results*: `submit` (returns a `Future`), `invokeAll`/`invokeAny`, and shutdown methods. `ScheduledExecutorService` adds time: `schedule`, `scheduleAtFixedRate` (fixed period, cron-like), and `scheduleWithFixedDelay` (delay measured *after* each run completes). A subtle trap with `scheduleAtFixedRate`: if a run overruns its period, executions do not run concurrently — they queue up and bunch, because a single scheduled task is never run by two threads at once.

`submit` wraps your `Runnable`/`Callable` in a `FutureTask`. `Callable<V>` is the value-returning, checked-exception-throwing cousin of `Runnable`; `Future.get()` blocks until completion and **re-throws the task's exception wrapped in `ExecutionException`**. The senior pitfall: exceptions from tasks submitted via `submit` are *swallowed into the Future* — if you never call `get()`, the failure vanishes silently. Tasks handed to `execute` instead propagate to the thread's `UncaughtExceptionHandler`.

## ThreadPoolExecutor Internals

`Executors.newFixedThreadPool(...)` and friends are thin wrappers over `ThreadPoolExecutor`, whose constructor exposes the real knobs: **corePoolSize, maximumPoolSize, keepAliveTime, workQueue, threadFactory, rejectedExecutionHandler**. The scheduling algorithm is counterintuitive and the source of most production incidents:

```
task arrives
 ├─ threads < core?        → start a new core thread
 ├─ else queue.offer() ok? → enqueue (do NOT grow the pool)
 ├─ else threads < max?    → start a new (non-core) thread
 └─ else                   → reject
```

The critical insight: **the pool only grows past `corePoolSize` when the queue rejects an offer (is full)**. So `newFixedThreadPool`, which uses an *unbounded* `LinkedBlockingQueue`, can never reach `maximumPoolSize` and keep-alive never triggers — the queue always accepts, so the pool sits at core size while the queue grows without bound (a slow OOM under overload). Conversely `newCachedThreadPool` uses a `SynchronousQueue` (zero capacity: an `offer` succeeds only if a thread is waiting), so every burst spawns threads up to `Integer.MAX_VALUE` — a thread bomb. Correct production config is usually an explicit `ThreadPoolExecutor` with a **bounded** queue plus a deliberate rejection policy.

The four `RejectedExecutionHandler`s: `AbortPolicy` (default, throws `RejectedExecutionException`), `CallerRunsPolicy` (runs the task on the submitting thread — a natural backpressure valve that slows producers), `DiscardPolicy`, and `DiscardOldestPolicy`. `CallerRunsPolicy` is the pragmatic choice when you want the system to self-throttle rather than drop or explode.

## Lifecycle and Graceful Shutdown

A pool moves through `RUNNING → SHUTDOWN → STOP → TIDYING → TERMINATED`. `shutdown()` is graceful — it stops accepting new tasks but drains the queue; `shutdownNow()` attempts to `interrupt()` running tasks and returns the undrained queue. The correct pattern is shutdown, then `awaitTermination(timeout)`, then `shutdownNow` if it lingers. Since Java 19 `ExecutorService` is `AutoCloseable`, so try-with-resources does exactly this dance (close = shutdown + await, escalating to shutdownNow on interrupt) — but note `close()` blocks the current thread until tasks finish, which can surprise you.

## Pool Sizing: CPU- vs I/O-Bound

Sizing is a modeling problem. For **CPU-bound** work the pool should be roughly `N_cpu` (or `N_cpu + 1` to cover the occasional page fault) — more threads only add context-switch and cache-thrash overhead with no throughput gain. For **I/O-bound** work threads spend most time parked, so you want many more; Goetz's formula is `N_threads = N_cpu × U × (1 + W/C)`, where `U` is target utilization and `W/C` is the wait-to-compute ratio. This formula is precisely the pain point that **virtual threads** dissolve: instead of computing `W/C` and sizing a scarce pool, you spawn one cheap thread per task and let blocking be free. Pools do not vanish — they remain correct for bounding *CPU-bound* parallelism and for capping a scarce resource (DB connections) — but they stop being the default unit of concurrency for blocking work.

## See Also

- [[threads-and-the-os]]
- [[fork-join-and-parallel-streams]]
- [[locks-and-atomics]]
- [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]]
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]]
- [[virtual-threads]] — the modern successor for I/O-bound tasks
