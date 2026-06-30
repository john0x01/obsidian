# Promises Internals

A promise is a state machine for a value that does not exist yet, plus a contract about *when* its callbacks run. The value most engineers underuse is the precise vocabulary the spec defines — states vs fates, resolution vs fulfillment, jobs vs the queue. Getting that vocabulary exact dissolves most "why did this log in that order?" confusion.

## States and Fates — Two Different Axes

A promise has one of three **states**: `pending`, `fulfilled`, or `rejected`. Orthogonally it has a **fate**: `unresolved` or `resolved`. These are not the same axis, and conflating them is the root of much confusion.

- *Fulfilled* and *rejected* are the two **settled** states; they are terminal and immutable. Once settled, a promise never changes value or state again.
- *Resolved* (the fate) means the promise has been "locked in" to follow something — either a concrete value or *another* thenable. A resolved promise can still be **pending** if it was resolved *to another pending promise*; its state will track that target.

So "resolved" ≠ "fulfilled." `resolve(anotherPendingPromise)` produces a promise that is resolved-but-pending. This distinction is exactly what the next section is about.

## Resolution vs Fulfillment — Thenable Assimilation

`resolve(x)` does not blindly fulfill with `x`. The spec runs the **Resolve Promise** procedure:

- If `x` is a thenable (any object with a callable `.then`), the promise *assimilates* it: it calls `x.then(...)` and adopts `x`'s eventual settlement. The outer promise stays pending until `x` settles, then mirrors it.
- If `x` is a non-thenable value, the promise fulfills with `x`.

This is why returning a promise from a `.then` handler "flattens" instead of nesting: the chained promise resolves *to* the returned promise and follows it. Two consequences seniors should hold:

1. A promise can never fulfill *with* a native promise — assimilation always unwraps one level (and chains further if the result is itself thenable).
2. `reject(x)` does **not** assimilate: rejecting with a thenable rejects with that object as-is. Rejection is value-faithful; resolution is value-adopting.

## The Microtask Queue and Job Scheduling

When a promise settles, its registered reactions are not called synchronously. The spec schedules them as **jobs** (`PromiseReactionJob`). Hosts implement the job queue as the **microtask queue**, which the event loop drains *completely* after each macrotask and *before* the next render or timer callback. The guarantees that matter:

- A `.then` callback **always** runs asynchronously, on a future microtask, even if the promise is already settled when you attach it. This is the "never release Zalgo" rule — a promise is never sometimes-sync, sometimes-async.
- Each chained `.then` is its own job: a long chain spreads across multiple microtask turns, interleaving with other drained microtasks but never with macrotasks or rendering.

```
macrotask (e.g. script / timer)
  └─ run sync code
  └─ drain ALL microtasks (promise reactions, queueMicrotask)
       └─ reactions may enqueue more microtasks → drained too
  └─ (render) → next macrotask
```

Microtasks enqueued *during* draining are also drained in the same turn — an unbounded microtask loop can starve rendering and timers entirely.

## then / catch / finally Semantics

- `then(onF, onR)` returns a **new** promise. Its fate depends on the handler: a returned value fulfills it (after assimilation), a thrown error rejects it, a returned thenable is followed.
- `catch(fn)` is exactly `then(undefined, fn)`. With no matching handler, fulfillment/rejection passes through to the next link — this *pass-through* is what makes a single trailing `.catch` cover an entire chain.
- `finally(fn)` is value-transparent: it runs `fn` for side effects, ignores its return value, and forwards the *original* settlement — **unless** `fn` throws or returns a rejecting promise, which overrides the outcome. It also waits if `fn` returns a thenable.

## Combinators

- `Promise.all(iter)` — fulfills with an ordered array when *all* fulfill; rejects on the *first* rejection (others keep running but are ignored). Fail-fast.
- `Promise.allSettled(iter)` — never rejects; resolves with `{status, value|reason}` per input. Use when you want every outcome.
- `Promise.race(iter)` — settles as the first input to **settle**, fulfilled *or* rejected.
- `Promise.any(iter)` — fulfills with the first *fulfillment*; rejects only if *all* reject, with an `AggregateError`. The mirror image of `all`.

A subtle senior point: `all`/`race` do not cancel the losers — promises have no cancellation. "Discarded" branches still run to completion and can still throw, becoming unhandled rejections.

## Error Propagation and Unhandled Rejections

A thrown error or `reject` propagates *down* the chain until a rejection handler catches it, analogous to synchronous `try/catch` but across microtask turns. If a rejected promise reaches the end of microtask draining with **no** rejection handler attached, the host fires the `unhandledrejection` event (browser) / `unhandledRejection` (Node). Because handlers can be attached *later*, the host defers this judgment to the end of the turn; attaching `.catch` after the fact emits `rejectionhandled`. The classic leak: a promise stored but never `.catch`-ed, or a forgotten `await`, turns a bug into a silent process-level warning rather than a local exception.

## See Also
- [[async-await]]
- [[generators-as-coroutines]]
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
- [[10-frameworks-and-stacks/node-js/architecture/timers-and-microtasks|Node Timers & Microtasks]] — Node's queue ordering
