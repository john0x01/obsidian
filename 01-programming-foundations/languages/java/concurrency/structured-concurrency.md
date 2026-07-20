# Structured Concurrency

Structured concurrency (the `StructuredTaskScope` API, plus *scoped values* as the [[virtual-threads|virtual-thread]]-friendly replacement for `ThreadLocal`) treats a group of concurrent subtasks as a **single unit of work with a bounded lifetime**. It solves the problem that a bare `ExecutorService` + `Future` gives you no relationship between tasks: work leaks, errors get swallowed, and cancellation is a manual, error-prone afterthought. Note on status: virtual threads are final, but structured concurrency spent 21→25 in *preview* with more than one API redesign, and scoped values finalized only late in that line — treat the APIs below as the shape, and verify the exact surface against your JDK.

## The Core Principle: Concurrency Follows Code Structure

Structured programming replaced `goto` by insisting control flow nest — a block is entered once and exited once. Structured concurrency applies the same discipline to threads: **a subtask's lifetime is bounded by a lexical scope, and the parent cannot exit that scope until every child has finished, failed, or been cancelled.** Concurrency thus nests exactly like method calls, giving you back the invariants that unstructured spawning destroys:

- **No leaks.** A running task can never outlive the block that created it. There are no orphans.
- **Error propagation.** A subtask failure surfaces at the join point as an exception, not as a value silently trapped in a `Future` nobody reads.
- **Cancellation propagation.** If one branch fails (or the whole scope is interrupted), siblings are automatically cancelled via interruption — no manual `future.cancel()` bookkeeping.
- **Observability.** Because the parent–child relationship is real, thread dumps and profilers can show the task *tree*, not a flat soup of pool threads.

## The Shape of the API

Each `fork` spawns a cheap virtual thread; the scope is used with try-with-resources so its lifetime is unmistakable. The classic pattern joins concurrent subtasks and fails fast if any fails:

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user  = scope.fork(() -> fetchUser(id));      // each = a virtual thread
    var order = scope.fork(() -> fetchOrder(id));
    scope.join();                 // wait for ALL forks
    scope.throwIfFailed();        // propagate the first failure, cancel siblings
    return new Page(user.get(), order.get());
}   // scope closes: any still-running fork is guaranteed cancelled
```

`ShutdownOnFailure` is fail-fast fan-out/fan-in; `ShutdownOnSuccess` is the racing dual — the first successful subtask cancels the rest (a hedged request, "fastest replica wins"). Hedge/verify: later previews reworked this into a factory-plus-*joiner* design (roughly `StructuredTaskScope.open(...)` with a `Joiner` policy) that supersedes the `Shutdown*` subclasses — so confirm the names in the Java 25 javadoc before relying on them. The *concept* — fork, single join, policy-driven shutdown, guaranteed cleanup on close — is stable regardless of the surface.

## Contrast with Unstructured Executor Code

The pre-Loom idiom is `goto`-like:

```java
Future<User>  u = pool.submit(() -> fetchUser(id));   // fire-and-forget shape
Future<Order> o = pool.submit(() -> fetchOrder(id));
User user = u.get();   // if this throws, o keeps running — leaked and orphaned
```

Nothing ties `u` and `o` together or to this method. If `u.get()` throws, `o` runs on invisibly, consuming a connection; its eventual exception may never be observed. Cancellation is opt-in and racy. The lifetimes escape the call stack — the exact defect structured programming outlawed for sequential code, reappearing in the concurrent dimension. Contrast this with JavaScript's `Promise.all`, which at least *aggregates* results but still cannot cancel the losers; structured concurrency's guaranteed sibling cancellation on scope exit goes further.

## Scoped Values: Immutable Context Down the Task Tree

`ThreadLocal` is unsuited to millions of virtual threads: it is mutable, inheritance is leaky, `remove()` is easily forgotten, and per-thread maps balloon. **Scoped values** are the replacement: an immutable binding established for the *dynamic extent* of a lambda and automatically, cheaply inherited by structured subtasks.

```java
private static final ScopedValue<User> CURRENT = ScopedValue.newInstance();

ScopedValue.where(CURRENT, user).run(() -> {
    // anywhere in this call tree, incl. forked subtasks:
    var u = CURRENT.get();          // no setter — rebinding needs a nested where()
});   // unbound automatically here — no leak, no cleanup call
```

The binding is one-way (parent → children), immutable, and torn down when the scope exits, so it composes safely with unbounded virtual threads. Scoped values appear to have finalized late in the 21→25 line (verify the JEP/release for your JDK).

## Senior Pitfalls and Philosophy

- **Don't hide a scope inside a long-lived object** or return a live `Subtask`; that smuggles concurrency back out of the block and reintroduces leaks.
- **`join()` is mandatory** before reading results — reading a subtask before join is a programming error.
- **Interruption is still cooperative** — subtasks doing tight CPU loops must poll for interruption, or cancellation cannot take effect.
- **Rebinding, not mutation** — a subtask needing a different context value opens its own nested `where(...)`, keeping the value effectively immutable.

The philosophy mirrors the language shift: `try/catch/finally` for a single stack, `StructuredTaskScope` for a *tree* of stacks. Failure, cancellation, and cleanup become compositional properties of a block rather than discipline you must remember to enforce.

## See Also

- [[virtual-threads]]
- [[completablefuture-and-async]]
- [[exceptions-and-error-handling]]
- [[03-computer-systems/concurrency-and-parallelism/csp-and-channels|CSP and Channels]]
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]]
