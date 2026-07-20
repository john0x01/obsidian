# Volatile and Synchronized

`volatile` and `synchronized` are the two primitive synchronization mechanisms the Java Memory Model gives you to create happens-before edges. They are constantly confused because both touch "visibility," but they solve different problems: `volatile` provides **visibility + ordering** for a single variable; `synchronized` provides **mutual exclusion + visibility** for arbitrary compound operations. Choosing wrongly produces code that is either subtly broken or needlessly slow.

## Volatile: Visibility And Ordering, Not Atomicity

Declaring a field `volatile` makes every read fetch from main memory and every write flush to it, but the deeper guarantee is the [[java-memory-model|happens-before]] edge: **a write to a volatile field happens-before every subsequent read of that field**. The JMM also forbids reordering across a volatile access — earlier writes cannot be moved after a volatile write, and later reads cannot be moved before a volatile read. This is exactly an acquire/release pairing at the language level (see [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]).

What `volatile` does *not* give you is **atomicity for compound actions**. The classic trap:

```java
volatile int counter;
counter++;   // read, increment, write — three actions, NOT atomic
```

Two threads can both read the same value and both write back the same increment, losing an update. `volatile` guarantees each *individual* read and write is visible and ordered; it cannot make a read-modify-write indivisible. For that you need a lock or an atomic class (`AtomicInteger`, `VarHandle` CAS) — see [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]] and [[locks-and-atomics]]. The correct use of `volatile` is for **flags and single-writer/single-publisher references**, where no compound action exists.

## Synchronized: Mutual Exclusion Plus Visibility

`synchronized` acquires the **monitor** associated with an object — conceptually a lock stored in the object's header (the *mark word*). It provides two things at once:

- **Mutual exclusion** — only one thread holds a given monitor at a time, making the guarded block atomic with respect to other synchronized blocks on the same monitor.
- **Visibility** — **a monitor exit (unlock) happens-before every subsequent monitor enter (lock) on the same monitor.** So everything a thread wrote inside the block is visible to the next thread that enters.

Two further properties matter. Monitors are **reentrant**: a thread already holding a monitor can re-acquire it (the lock counts depth), so a synchronized method calling another synchronized method on `this` does not self-deadlock. And the monitor is per-object: synchronizing on different objects gives no exclusion between them. Historically HotSpot used *biased locking* to cheapen uncontended monitor entry, but that optimization was deprecated and removed in modern HotSpot; do not design around it. The general theory of locks lives in [[03-computer-systems/concurrency-and-parallelism/mutexes-and-locks|Mutexes and Locks]].

## The Precise Difference

| | `volatile` | `synchronized` |
|---|---|---|
| Visibility | yes | yes |
| Ordering | yes | yes |
| Mutual exclusion / atomicity | **no** | **yes** |
| Granularity | one field | arbitrary block |
| Blocking | never | yes (may contend) |

Rule of thumb: if a single thread ever needs to *read-then-write* shared state, you need exclusion (`synchronized` or an atomic). If you only publish a value that another thread reads, `volatile` suffices and is cheaper.

## Double-Checked Locking And Why The Field Must Be Volatile

The canonical case that ties both together is the lazy singleton:

```java
class Holder {
    private volatile Instance instance;   // volatile is mandatory
    Instance get() {
        Instance i = instance;
        if (i == null) {
            synchronized (this) {
                i = instance;
                if (i == null) instance = i = new Instance();
            }
        }
        return i;
    }
}
```

Without `volatile`, the fast path reads `instance` *outside* the lock, so it gets no happens-before edge from the constructing thread. The constructor's field writes and the assignment of the reference can be reordered: a second thread could observe a **non-null reference to a partially-constructed object** — the unsafe-publication hazard detailed in [[safe-publication]]. The `volatile` write to `instance` publishes the fully-constructed object and orders the constructor stores before it; the matching `volatile` read on the fast path completes the edge. The local `i` is a micro-optimization avoiding a second volatile read.

## Senior Pitfalls

- Using `volatile` where you need atomicity (`v++`, check-then-act) — silent lost updates.
- Synchronizing on a mutable or shared-interned object (`String` literals, boxed primitives) — accidental cross-talk on the same monitor.
- Assuming `volatile` on an *array reference* makes element accesses volatile — it does not; the elements are plain.

## See Also

- [[java-memory-model]]
- [[safe-publication]]
- [[locks-and-atomics]]
- [[03-computer-systems/concurrency-and-parallelism/mutexes-and-locks|Mutexes and Locks]]
- [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]]
