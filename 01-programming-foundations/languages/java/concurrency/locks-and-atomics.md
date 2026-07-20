# Locks and Atomics

Above `synchronized` and `volatile` (see [[volatile-and-synchronized]]) sits `java.util.concurrent.locks` and `j.u.c.atomic` — explicit locks with capabilities the intrinsic monitor lacks, and lock-free atomics for the hot paths where even an uncontended lock is too much. Understanding them means understanding the one class they nearly all share underneath: **AQS**.

## ReentrantLock and What synchronized Cannot Do

`ReentrantLock` gives the same mutual exclusion and happens-before semantics as `synchronized`, but adds what a block-structured monitor cannot: `tryLock()` (non-blocking or with a timeout — the standard deadlock-avoidance tool), `lockInterruptibly()` (respond to cancellation while waiting), an optional **fairness** policy (FIFO grants, trading throughput for no starvation), and *non-nested* locking (acquire in one method, release in another). The cost is discipline: you **must** `unlock()` in a `finally`, or an exception leaks the lock forever — exactly the footgun `synchronized` eliminates by releasing on scope exit. Rule of thumb: prefer `synchronized` for simple mutual exclusion; reach for `ReentrantLock` only when you need one of its extra capabilities. (Historically `synchronized` also *pinned* virtual threads to their carrier; modern HotSpot has removed that limitation, narrowing the old reason to prefer `ReentrantLock` under Loom.)

`ReadWriteLock` (`ReentrantReadWriteLock`) splits access into a shared **read** lock (many concurrent readers) and an exclusive **write** lock. It wins only for genuinely read-heavy workloads where read critical sections are non-trivial; otherwise the bookkeeping costs more than a plain lock. You can **downgrade** (hold write, acquire read, release write) but never **upgrade** read→write — two readers both trying to upgrade deadlock.

`StampedLock` goes further with an **optimistic read** mode that is not a lock at all: `tryOptimisticRead()` returns a version stamp, you read the fields, then `validate(stamp)` checks no writer intervened; if it did, you fall back to a real read lock. Because the optimistic path issues no CAS and no writes to lock state, it is dramatically cheaper under read-mostly contention. The catch: `StampedLock` is **not reentrant** and supports no `Condition`s — re-acquiring it on the same thread self-deadlocks.

`Condition` replaces `Object.wait/notify` for explicit locks: `await`/`signal`/`signalAll`, obtained via `lock.newCondition()`. Its advantage is **multiple wait-sets per lock** — a bounded buffer can have separate `notFull` and `notEmpty` conditions so producers and consumers wake only the right waiters (see [[03-computer-systems/concurrency-and-parallelism/condition-variables|Condition Variables]]). As with `wait`, always `await` in a `while` loop guarding the predicate, never an `if` — spurious wakeups are permitted.

## AQS: The Shared Machinery

`ReentrantLock`, `Semaphore`, `CountDownLatch`, and `ReentrantReadWriteLock` are all thin skins over **`AbstractQueuedSynchronizer`**. AQS holds a single `volatile int state` and a FIFO wait queue (a CLH-derived linked list). Subclasses define what `state` *means* — for `ReentrantLock` it's the reentrancy hold count; for `Semaphore` it's permits; for `CountDownLatch` it's the countdown — by implementing `tryAcquire`/`tryRelease` (exclusive) or the `...Shared` variants. AQS handles the hard part: it CASes `state`, and on failure enqueues the thread and parks it via `LockSupport.park` (which maps to OS futex/`unpark`). This is why every j.u.c synchronizer shares identical fairness, interruption, and timeout behavior — it is all one battle-tested state machine, and knowing it lets you read any of these classes' source fluently.

## Atomics, CAS, and the ABA Problem

`AtomicInteger`, `AtomicLong`, and `AtomicReference` provide lock-free updates built on **compare-and-swap** — `compareAndSet(expected, new)` compiles to a single CPU instruction (`CMPXCHG` on x86) that atomically swaps only if the current value still equals `expected`. The idiom is a retry loop: read, compute, CAS, repeat on failure. CAS carries the same acquire/release memory semantics as a `volatile` access, so it also publishes safely — the bridge to [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]] and to the fine-grained modes (`getAcquire`, `setRelease`, `getPlain`) exposed via `VarHandle`. This is the concrete embodiment of [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]].

CAS has a famous blind spot: the **ABA problem**. A thread reads value `A`, and before its CAS another thread changes `A→B→A`. The CAS succeeds because the value *looks* unchanged, yet the world moved underneath it (e.g., a reused node in a lock-free stack). The fix is to version the reference: `AtomicStampedReference` pairs the value with a monotonically increasing stamp so the CAS validates both.

Under heavy write contention, a single `AtomicLong` becomes a hotspot — every core fights to CAS the same cache line, and failed retries burn cycles. `LongAdder` (and `DoubleAdder`) solve this by **striping**: writes scatter across an array of `Cell`s (dynamically grown on contention), and `sum()` adds them up. The trade-off is deliberate — you gain write throughput but lose a consistent instantaneous snapshot (`sum()` is not atomic across cells) and spend more memory. Use `LongAdder` for high-frequency counters (metrics, statistics); use `AtomicLong` when you need an exact, atomically-readable value or CAS-based logic.

## See Also

- [[volatile-and-synchronized]]
- [[java-memory-model]]
- [[concurrent-collections]]
- [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]
- [[03-computer-systems/concurrency-and-parallelism/mutexes-and-locks|Mutexes and Locks]]
- [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]]
- [[varhandle-and-unsafe]] — low-level atomic access modes
