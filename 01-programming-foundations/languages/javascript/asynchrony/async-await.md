# Async Await

`async`/`await` is not a new concurrency primitive. It is *syntactic sugar* over promises and the same suspend/resume machinery that powers generators, compiled by the engine into a state machine. Reading it that way — as a coroutine that parks itself on the microtask queue — explains every one of its surprising behaviors, from ordering to the sequential-await performance trap.

## What `async` and `await` Actually Mean

- An `async` function **always returns a promise**. A plain `return v` resolves it with `v` (assimilating if `v` is thenable); a `throw` rejects it. The body runs synchronously *up to the first `await`*, then returns the still-pending promise to the caller.
- `await expr` does two things: it `resolve`s `expr` (assimilating thenables exactly like `Promise.resolve`), and it **suspends** the function until that promise settles, then resumes with the fulfillment value — or *throws* the rejection reason at the await point.

The key reframing: `await` is a *yield point*. The function is a coroutine that hands control back to the event loop and schedules its own continuation as a microtask.

## Desugaring to a State Machine

Conceptually the engine rewrites an async function into a generator driven by a runner. Each `await` becomes a `yield`; a driver (`Promise.resolve(yielded).then(resume)`) re-enters the generator when the awaited promise settles, passing the value back in. The historical proof of this is that `async`/`await` was polyfilled for years by exactly this transform (Babel's `regeneratorRuntime`, TypeScript's `__awaiter`/`__generator`):

```js
async function f() {
  const a = await g();   // suspend point 1
  return a + (await h()); // suspend point 2
}
// ≈ driven generator:
function* f_gen() {
  const a = yield g();
  return a + (yield h());
}
// driver: each yield → Promise.resolve(val).then(v => gen.next(v))
//         a rejection → gen.throw(reason)  (so try/catch works)
```

The function's locals, the current await index, and `this` are captured as the **state machine's state**, persisted across suspensions on the heap. Resumption restores that frame and continues. This is why local variables survive an `await` even though the call stack unwound entirely in between.

## How Suspension and Resumption Use Microtasks

When you hit `await p`:

1. The function's call stack frame **unwinds** — control returns to whatever called the async function (or to the event loop).
2. A reaction is registered on `p`. When `p` settles, that reaction is scheduled as a **microtask**.
3. On the next microtask drain, the engine **resumes** the function, rebuilding its frame, with the resolved value (or throwing the reason).

Two consequences. First, even `await Promise.resolve(0)` defers the rest of the function to a future microtask — code *after* the first await never runs in the same synchronous turn. Second, because resumption is a microtask, an awaiting function interleaves with other microtasks but always runs before timers and rendering. This ties `async`/`await` directly to the microtask queue described in the promises and runtime notes.

## Sequential vs Concurrent — The Defining Pitfall

`await` *serializes*. Each `await` fully completes before the next statement, so awaits in sequence run their operations back-to-back, not in parallel:

```js
// SEQUENTIAL — ~ t(a) + t(b): b starts only after a resolves
const a = await fetchA();
const b = await fetchB();

// CONCURRENT — ~ max(t(a), t(b)): both started, then awaited
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

The trap deepens inside loops: `for (const x of xs) await work(x)` runs strictly one at a time. To fan out, *start* the promises first (`xs.map(work)`) then `await Promise.all(...)`. The rule of thumb: **`await` where you need the result, not where you start the work.** Conversely, kicking off a promise without awaiting it (`work()` with no `await`) creates a floating promise whose rejection becomes unhandled.

## Error Handling

Because a rejection is *thrown* at the await point, ordinary `try/catch/finally` works and is the idiomatic form. Caveats:

- A floating (un-awaited) promise's rejection escapes the surrounding `try` — there is no await point for it to throw at, so it becomes an unhandled rejection.
- In `Promise.all`, the first rejection throws at your `await`; the other operations still run to completion (no cancellation).
- `await` in a `finally` block, or after the `try`, can swallow or reorder errors in subtle ways; keep cleanup synchronous where possible.

## Relation to the Event Loop

An async function is cooperative: it only yields at `await`. Synchronous work *between* awaits still blocks the single thread completely — `async` does not move CPU work off-thread. It restructures *when* code runs (parked on microtasks) without adding parallelism. For true parallelism you still need Workers; `async`/`await` is about non-blocking I/O orchestration, not multithreading.

## See Also
- [[promises-internals]]
- [[generators-as-coroutines]]
- [[module-systems]]
- [[03-computer-systems/concurrency-and-parallelism/async-and-await|Async and Await (Concurrency)]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
