# Safe Publication

**Safe publication** is the discipline of making an object visible to other threads such that they see it *fully constructed*. It is the operational complement to the [[java-memory-model|Java Memory Model]]: the JMM tells you what happens-before edges exist; safe publication tells you which idioms actually create the edge between "I finished building this object" and "another thread reads it." Getting this wrong is one of the most insidious concurrency bugs because the object *looks* complete in single-threaded testing.

## Publication Versus Escape

To **publish** an object is to make a reference to it available outside its current scope — storing it in a shared field, returning it, putting it in a collection, passing it to other threads. **Escape** is publication that happens *too early or too unsafely*: the object becomes reachable before it is fully built, or through a channel that provides no memory ordering. The most dangerous escape is leaking `this` from a constructor — see [[final-field-semantics]].

## The Unsafe-Publication Hazard

Consider publishing through a plain field:

```java
class Holder { Resource resource; }   // non-volatile

// Thread A
holder.resource = new Resource(/* sets up internal state */);

// Thread B
holder.resource.use();   // may see a half-built Resource
```

Two reorderings conspire. First, the constructor's internal writes and the assignment `holder.resource = ...` are not ordered with respect to a *racing* reader — Thread B can observe the reference *before* the constructor's field stores. Second, B's read of `resource` and its read of `resource`'s internals can themselves be reordered on weak hardware. The result: B sees a non-null reference but a **partially-constructed object** with default-valued fields. This is not a hypothetical; it manifests on weakly-ordered CPUs (ARM/POWER). The underlying class of bug is covered in [[03-computer-systems/concurrency-and-parallelism/race-conditions|Race Conditions]].

## The Safe-Publication Idioms

An object is *safely published* when the publishing write and the reading read are connected by a happens-before edge. The JMM guarantees this for exactly these channels:

- **Static initializer.** Storing the reference in a `static` field initialized at class-load time. The JVM serializes class initialization under a lock, providing the edge — the basis of the lazy *holder class* idiom.
- **`volatile` field (or `AtomicReference`).** The volatile/atomic write happens-before any subsequent read of that field; constructor stores precede it by program order, so they ride across the edge. (See [[volatile-and-synchronized]] and [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]].)
- **`final` field.** Reachable from a properly-constructed object's `final` fields — the freeze guarantee of [[final-field-semantics]]. This is the only idiom that protects against *racy* reads.
- **Lock-guarded field.** Writing and reading the field while holding the same lock; monitor exit happens-before the next monitor enter.

A useful shortcut: putting an object into a properly synchronized concurrent collection (e.g. `ConcurrentHashMap`, a `BlockingQueue`) safely publishes it, because those classes establish the edge internally.

## Immutable, Effectively Immutable, And Mutable State

Publication requirements scale with mutability:

- **Immutable objects** (all fields `final`, no `this` escape, deeply non-mutating) can be published through **any** mechanism, even a data race — the freeze rule does the work. This is the payoff of [[01-programming-foundations/paradigms/immutability|immutability]].
- **Effectively immutable objects** — technically mutable but never modified after publication — are thread-safe **if** they are *safely* published. The safe-publication edge substitutes for the freeze you'd get from `final`.
- **Mutable objects** must be **both** safely published **and** subsequently guarded by synchronization on *every* access. Safe publication only ensures the *initial* state is seen correctly; ongoing mutations each need their own happens-before coverage.

```
Object kind          Publish how                  Access how
immutable            any (even a race)            no sync
effectively imm.     safe publication required    no sync after
mutable              safe publication required    sync every access
```

## Senior Pitfalls

- **Safely publishing a mutable object and then mutating it without locks.** Publication fixed the first read, not the later ones — a common and silent error.
- **Relying on `synchronized` only on the writer.** If the reader reads the published field outside the lock, there is no edge (this is precisely the double-checked-locking trap).
- **Assuming construction order plus a plain assignment is enough on x86.** It often works on strong TSO hardware and breaks on ARM — non-portable, untestable bugs.
- **Leaking `this` during construction** (callbacks, thread starts), which defeats *every* publication strategy because the object escapes before it is built.

## Design Philosophy

Safe publication reframes concurrency as a question of *boundaries*: state crosses from one thread's exclusive ownership to shared visibility at exactly one well-defined point, and that crossing must carry a memory-ordering edge. Prefer immutable objects so the boundary is free; where mutability is unavoidable, confine the object to one thread or guard it consistently. The goal is to make the moment of sharing explicit and rare.

## See Also

- [[final-field-semantics]]
- [[java-memory-model]]
- [[volatile-and-synchronized]]
- [[03-computer-systems/concurrency-and-parallelism/race-conditions|Race Conditions]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
