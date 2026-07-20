# The Java Memory Model

The Java Memory Model (JMM), specified by JSR-133 in Java 5, is the contract that defines **when a write performed by one thread is guaranteed to be visible to a read performed by another**. It exists because, absent such a contract, the optimizations performed by compilers, the JIT, and the CPU would make multithreaded behavior unpredictable and unportable. The JMM is the abstract bridge between the source-level program you wrote and the wildly reordered machine instructions that actually execute.

## Why A Memory Model Is Necessary At All

Naively you might expect that a value stored to a field is immediately seen by every other thread. It is not. Three independent layers reorder and cache memory operations:

- The **compiler/JIT** may hoist, sink, eliminate, or reorder loads and stores as long as single-threaded semantics are preserved.
- The **CPU** executes out of order and buffers stores in per-core **store buffers**, so a store may sit invisible to other cores for an indeterminate time.
- The **cache hierarchy** means each core may hold a stale copy of a line.

Crucially, the JMM does **not** mention store buffers, cache coherence, or any specific chip. It is an *abstract* contract expressed purely in terms of program actions and an ordering relation, precisely so that the *same* Java program is correct on x86 (a strong TSO model) and on ARM/POWER (weak models). The hardware-level reordering rules live in [[03-computer-systems/computer-architecture/memory-models|Memory Models]]; the JMM sits one layer above and translates them into a portable promise.

## Happens-Before: The Central Relation

The JMM's core abstraction is **happens-before** (HB), a partial order over actions. If action A happens-before action B, then A's effects (its writes) are guaranteed visible to B. The edges that establish HB:

- **Program order** — within a single thread, each action HB every action that follows it in source order. (This does *not* mean no reordering occurs; it means reordering cannot be *observed* within that thread.)
- **Monitor lock** — an unlock of a monitor HB every subsequent lock of the *same* monitor.
- **Volatile** — a write to a `volatile` field HB every subsequent read of that same field.
- **Thread start** — `Thread.start()` HB every action in the started thread.
- **Thread join** — every action in a thread HB the return of `Thread.join()` on it.
- **Transitivity** — if A HB B and B HB C, then A HB C.

Transitivity is what makes the model usable: a non-volatile write that *precedes* a volatile write (in program order) becomes visible to a reader that *follows* the matching volatile read. This is the mechanism behind the publication idioms in [[safe-publication]]. The concrete machinery — fences, acquire/release — is covered in [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]].

```
Thread A                 Thread B
data = 42;     ─┐
ready = true;   │ (ready is volatile)
                └──HB──▶ if (ready)       // volatile read
                            use(data);    // sees 42, via transitivity
```

## Data Races And The DRF-SC Guarantee

A **data race** is defined precisely: two accesses to the same non-`final`, non-`volatile` variable, **at least one a write**, **not ordered by happens-before**. A program with no data races is *correctly synchronized*.

The JMM's headline guarantee is **DRF-SC** (data-race-free implies sequentially consistent): if your program is correctly synchronized, it behaves *as if* executed by a single global interleaving of thread steps — the intuitive model. You then reason about correctness without ever thinking about reordering. The discipline of structuring code so HB edges cover every shared access is the whole game; see [[03-computer-systems/concurrency-and-parallelism/race-conditions|Race Conditions]].

For programs that *do* contain races, the JMM still forbids "out-of-thin-air" values (you can never read a value no thread ever wrote) to preserve memory safety and security, but otherwise the results are essentially unspecified and non-portable. The lesson: races are not "slightly buggy," they place you outside the model entirely.

## Mental Model And Pitfalls

Think of the JMM as defining *visibility edges*, not *timing*. HB says nothing about *when* (no wall-clock guarantee) — only that *if* B sees past the edge, it sees everything before A. Senior pitfalls:

- Assuming a plain `boolean` flag will eventually be seen by a polling thread — without `volatile` the JIT may hoist the read out of the loop, looping forever.
- Believing `synchronized` only matters for atomicity — it equally provides the *visibility* edge; dropping it loses both.
- Treating `volatile` as a mutex — it gives ordering/visibility but no mutual exclusion (see [[volatile-and-synchronized]]).

JavaScript's `SharedArrayBuffer` + `Atomics` adopt a directly analogous sequentially-consistent-for-DRF model — useful contrast in [[01-programming-foundations/languages/javascript/concurrency/shared-memory-and-atomics|Shared Memory and Atomics]].

## See Also

- [[volatile-and-synchronized]]
- [[safe-publication]]
- [[final-field-semantics]]
- [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]
- [[03-computer-systems/computer-architecture/memory-models|Memory Models]]
- [[01-programming-foundations/languages/javascript/concurrency/shared-memory-and-atomics|Shared Memory and Atomics]]
