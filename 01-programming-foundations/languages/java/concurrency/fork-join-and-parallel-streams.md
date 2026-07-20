# Fork/Join and Parallel Streams

`ForkJoinPool` is a specialized executor for **recursive divide-and-conquer** work: split a problem into subtasks, run them in parallel, join the results. Its defining feature is **work-stealing**, which keeps every core busy without a central task queue becoming a bottleneck. Parallel streams are the mass-market front-end to this machinery — and the place where most engineers accidentally hurt performance while believing they helped it.

## Work-Stealing Deques

An ordinary thread pool has one shared queue; every worker contends on the same lock to dequeue. `ForkJoinPool` instead gives **each worker its own double-ended queue (deque)**. A worker pushes and pops subtasks from the *head* of its own deque in LIFO order — LIFO because the most recently forked subtask is the hottest in cache and its data is still resident. When a worker's own deque empties, it becomes a thief: it **steals from the *tail*** of a random victim's deque, in FIFO order. Stealing the oldest task is deliberate — older tasks tend to be larger (nearer the root of the recursion), so one steal hands the thief a big chunk of work and amortizes the theft.

```
worker A deque:  [ big ...... small ]
                   ^tail        ^head
 A works here ─────────────────────┘ (LIFO, hot cache)
 thief B steals ──┘ (FIFO, coldest & largest)
```

This design means contention scales with *idleness*, not throughput: busy workers touch only their own deque with no synchronization, and stealing happens only when a core would otherwise sit idle. It is a near-optimal load balancer for irregular workloads.

## RecursiveTask, RecursiveAction, and join Semantics

You express work as `RecursiveTask<V>` (returns a value via `compute()`) or `RecursiveAction` (void). The idiom: if the problem is below a threshold, compute directly; otherwise split, `fork()` one half (push to your deque), recurse on the other, then `join()`. The order matters — `fork()` the second half, compute the first inline, then `join`: this keeps the current thread productive instead of immediately blocking.

The magic is that `join()` is not a dumb block. If the joined subtask hasn't finished, the calling worker doesn't park uselessly — it **helps**, running that subtask (or others) itself. This is what prevents the pool from deadlocking when all threads are simultaneously waiting on children. The corollary pitfall: **never do blocking I/O inside a fork/join task**. A parked worker can't help, the pool doesn't compensate by default, and throughput collapses. If you must block, wrap it in a `ForkJoinPool.ManagedBlocker`, which lets the pool spin up a compensation thread to preserve target parallelism.

## The Common Pool and Its Trap

`ForkJoinPool.commonPool()` is a lazily-initialized, JVM-wide singleton whose default parallelism is `availableProcessors() - 1` (the submitting thread counts as one worker). **Parallel streams and most `CompletableFuture` async methods run here by default.** Because it is shared process-wide, one long-running or blocking task in the common pool starves *every other* parallel stream in the application — a classic latent coupling bug that only surfaces under load. The senior discipline: for anything that might block or run long, submit to a *dedicated* `ForkJoinPool` you own (submitting a task from within a custom pool makes the stream execute on that pool), never the common one.

## When Parallel Streams Help — and When They Hurt

`stream().parallel()` splits the source via `Spliterator.trySplit()` and maps the splits onto the common pool. It pays off only when the arithmetic works out: `N × Q` (elements times per-element cost) must dwarf the fixed overhead of splitting, task submission, and merging. Concretely, parallelism helps when:

- **N is large and Q is non-trivial** (CPU-bound, not a cheap field access).
- **The source splits cheaply and evenly** — arrays and `ArrayList` are `SIZED`/`SUBSIZED` and bisect in O(1); `LinkedList`, `Stream.iterate`, and I/O-backed streams split poorly or degrade to sequential.
- **Operations are stateless and side-effect-free**, so the combiner can merge partial results.

It hurts, often dramatically, when:

- **Boxing dominates** — use `IntStream`/`LongStream`, not `Stream<Integer>`; autoboxing allocation (see [[heap-and-allocation]]) can cost more than the parallelism saves.
- **There is shared mutable state** — writing to a shared collection in `forEach` is a data race; you must use a proper concurrent reduction (`collect` with an associative combiner, or a `Collector`).
- **Order matters** — `forEachOrdered` and ordered `limit` reimpose sequencing that erases the gains.
- **The workload is small**, where fixed overhead simply loses to a plain sequential loop.

The mental model: a parallel stream is a `ForkJoinPool` job in disguise. Reach for it only after you have measured, sized the split, eliminated boxing and shared state, and confirmed you are not poisoning the common pool.

## See Also

- [[executors-and-thread-pools]]
- [[locks-and-atomics]]
- [[heap-and-allocation]]
- [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]]
- [[03-computer-systems/concurrency-and-parallelism/parallelism-patterns|Parallelism Patterns]]
- [[03-computer-systems/concurrency-and-parallelism/amdahls-and-gustafsons-law|Amdahl's and Gustafson's Law]]
- [[stream-api]] — the pipeline API that parallelizes onto FJP
