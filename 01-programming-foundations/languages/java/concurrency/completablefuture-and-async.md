# CompletableFuture and Async

`CompletableFuture<T>` (Java 8) is a `Future` you can complete manually *and* compose — the JDK's tool for building non-blocking async pipelines out of dependent stages. It is Java's closest analogue to a JS Promise, and understanding it matters both for maintaining pre-Loom reactive code and for seeing precisely *why* [[virtual-threads|virtual threads]] let you delete most of it in favour of plain blocking code.

## Composing Pipelines

A `CompletableFuture` is a node in a dependency graph; each combinator registers a continuation that fires when the upstream stage completes. The core vocabulary, with its JS Promise cognate:

- `thenApply(fn)` — map: transform the value synchronously. Like `.then(v => ...)` returning a plain value.
- `thenCompose(fn)` — flatMap: `fn` returns *another* `CompletableFuture`, and the result is flattened rather than nested (`CF<CF<T>>` → `CF<T>`). This is exactly Promise **assimilation** — `.then` returning a thenable.
- `thenCombine(other, biFn)` — zip two independent futures into one. Like awaiting two promises and combining.
- `thenAccept` / `thenRun` — consume with side effects / run with no input.
- `allOf(cfs...)` / `anyOf(cfs...)` — fan-in combinators (see pitfalls). Analogues of `Promise.all` / `Promise.race`.
- `exceptionally(fn)` — recover from failure, like `.catch`.
- `handle(biFn)` / `whenComplete(biAction)` — see both value and throwable (`handle` transforms; `whenComplete` only peeks and re-throws).

```java
fetchUser(id)                                   // CF<User>
    .thenCompose(u -> fetchOrders(u))           // CF<List<Order>>, flattened
    .thenApply(this::summarize)                 // CF<Summary>
    .exceptionally(ex -> Summary.empty());      // recover
```

## The Executor Model — the Real Footgun

Every combinator has an `*Async` twin (`thenApplyAsync`, ...) and an overload taking an `Executor`. The distinction is subtle and bites hard:

- **Non-`Async`** stages run on *whatever thread completed the previous stage* — possibly the thread that called `complete()`, possibly a pool thread, possibly the caller inline. You do not control where.
- **`Async`** stages are submitted to an executor: the one you pass, or the **default** — `ForkJoinPool.commonPool()` (unless the JVM has one CPU, where a thread-per-task executor is used instead).

The default matters enormously. The common pool is a shared, CPU-sized, daemon pool also used by parallel streams. Doing **blocking** I/O on it starves the whole JVM's parallel work and silently caps throughput. Rule: for blocking stages, always pass an explicit executor; never block the common pool.

## The Sharpest Contrast with JS Promises: Zalgo

A JS promise reaction **always** runs on a future microtask, even if the promise is already settled — "never release Zalgo," a callback is never sometimes-sync/sometimes-async (see [[01-programming-foundations/languages/javascript/asynchrony/promises-internals|Promises Internals]]). `CompletableFuture` makes **no such guarantee.** If a stage is already complete when you attach a non-`Async` dependent, that dependent runs **synchronously, inline, on the calling thread**:

```java
var done = CompletableFuture.completedFuture(1);
done.thenApply(x -> { log("here"); return x; });  // runs "here" RIGHT NOW, on this thread
```

This is a genuine ordering hazard with no JS equivalent — reasoning about "what runs when" requires tracking completion timing, not just a single microtask queue. Other differences: CF stages can execute in true parallel on real threads (JS has one loop); there is no global microtask ordering — ordering is whatever the executors impose; and CF wraps failures in `CompletionException` (`join()`) or `ExecutionException` (`get()`), so you must `getCause()` to reach the original, unlike a promise rejection that carries the reason directly.

## Why Virtual Threads Make Most of This Optional

`CompletableFuture` exists to avoid **blocking a scarce platform thread**. Its whole design — decomposing logic into callback stages, `exceptionally` instead of `try/catch`, stack traces that show a pool thread rather than your logical call chain — is the price of never blocking. This is Java's version of callback hell, and the same "colored function" problem async/await papers over in JS.

Loom removes the premise. A blocking call on a virtual thread unmounts the carrier, so blocking is cheap again. The pipeline above collapses to:

```java
User user = fetchUser(id);          // blocks the VIRTUAL thread only; carrier is freed
List<Order> orders = fetchOrders(user);
return summarize(user, orders);     // real try/catch, real stack trace, top-to-bottom
```

For request-handling I/O orchestration, prefer this plus [[structured-concurrency]] over `CompletableFuture` chains. `CompletableFuture` remains genuinely useful where its model fits: manual completion bridging callback-style APIs (`complete`/`completeExceptionally`), declarative fan-out/fan-in via `allOf`, timeouts (`orTimeout`, `completeOnTimeout`), and interop with libraries that speak futures. It is a composition tool, not the default concurrency model.

## Senior Pitfalls

- **`allOf` discards results.** It returns `CompletableFuture<Void>`; you must re-read each input's value after it completes — unlike `Promise.all`, which hands you the array. Forgetting this yields `null`.
- **`cancel(true)` does not interrupt the work.** It only completes the future exceptionally with `CancellationException`; the underlying computation keeps running. CF cancellation is far weaker than `FutureTask` or a structured scope.
- **`join()` vs `get()`** — `join()` throws unchecked `CompletionException`; `get()` throws checked `ExecutionException`/`InterruptedException`. Pick deliberately and always unwrap `getCause()`.
- **Swallowed failures.** A chain with no `exceptionally`/`handle` whose result is never joined discards its exception silently — the CF analogue of an unhandled promise rejection.

## See Also

- [[virtual-threads]]
- [[structured-concurrency]]
- [[01-programming-foundations/languages/javascript/asynchrony/promises-internals|Promises Internals]]
- [[01-programming-foundations/languages/javascript/asynchrony/async-await|Async/Await]]
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]]
- [[10-frameworks-and-stacks/node-js/architecture/libuv-and-event-loop|Node Event Loop]]
