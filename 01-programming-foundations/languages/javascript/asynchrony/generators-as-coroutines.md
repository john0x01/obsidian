# Generators As Coroutines

A generator is JavaScript's one true **coroutine**: a function whose execution can be suspended at a `yield` and resumed later, preserving its entire local frame across the gap. Promises and `async`/`await` get most of the attention, but generators are the lower-level primitive — the *suspend/resume* mechanism that `async`/`await` is built on top of. Understanding them as cooperative coroutines, not just lazy iterators, is what unlocks the deeper async story.

## Suspend / Resume — Cooperative, Not Preemptive

Calling a generator function runs **none** of its body. It returns a *generator object* (an iterator) with its execution frozen at the top. Each `.next()` runs the body until the next `yield`, then **suspends** — saving the program counter, local variables, and `this` on the heap — and returns `{ value, done }`. Control returns to the caller; the frame persists, dormant.

This is *cooperative* concurrency: the generator decides when to yield control; nothing can preempt it mid-statement. Contrast OS threads, which the scheduler can interrupt at any instruction. A generator is single-threaded, deterministic, and yields only at explicit points — the same cooperative model the event loop relies on.

```js
function* counter() {
  let n = 0;
  while (true) {
    const reset = yield n++;   // suspends here, hands n out, waits for next()
    if (reset) n = 0;          // the value passed to next() reappears here
  }
}
const c = counter();
c.next();        // { value: 0, done: false }  — runs to first yield
c.next();        // { value: 1, done: false }  — resumes, runs to yield again
c.next(true);    // { value: 0, done: false }  — `reset` === true, n reset
```

## Two-Way Communication — `next(value)`, `throw`, `return`

The deep feature, and the one that makes generators *coroutines* rather than mere iterators, is that data flows **both ways**:

- `yield expr` sends `expr` *out* to the consumer (as `value`).
- `gen.next(v)` sends `v` *in*: it becomes the **evaluated result of the suspended `yield` expression** when the generator resumes. The first `.next()` cannot inject a value — there is no suspended `yield` yet to receive it.
- `gen.throw(err)` resumes the generator by *throwing* `err` at the yield point, so the generator's own `try/catch` can handle it.
- `gen.return(v)` forces an early completion, running any `finally` blocks.

This bidirectional channel is precisely the contract a promise-driver needs: yield a promise *out*, and when it settles, inject the resolved value *back in* via `next(value)` (or the rejection via `throw`). That is the entire mechanism behind `async`/`await`.

## Generators for Async Flow — The Historical Bridge

Before `async`/`await` was standardized, libraries used a generator + a **driver** to write asynchronous code that *looked* synchronous. You `yield` a promise; an external runner waits for it and resumes you with the result:

```js
function run(genFn) {
  const gen = genFn();
  function step(method, arg) {
    const { value, done } = gen[method](arg);   // next or throw
    if (done) return Promise.resolve(value);
    return Promise.resolve(value).then(
      v => step('next', v),                      // feed result back in
      e => step('throw', e)                      // throw error in
    );
  }
  return step('next');
}
```

co (the library) and TJ Holowaychuk's work popularized this; `async`/`await` is essentially this pattern *baked into the engine*, with `await` as a privileged `yield` and the driver built in. **redux-saga** still uses generators deliberately for a different reason: by yielding plain *effect descriptions* (objects describing "call this", "take that action") instead of promises, the saga middleware interprets the effects — making the async logic declarative and trivially testable by driving the generator with mock values, with no real I/O. The generator's two-way channel is what lets the test inject fake results.

## Delegation — `yield*`

`yield* iterable` delegates: it runs the inner iterable to exhaustion, forwarding each yielded value out and each injected `next(v)`/`throw` *in*, and finally evaluates to the inner generator's `return` value. It composes coroutines without manual relay loops — the building block for splitting a large generator into smaller ones, and for recursive traversals expressed as a flat stream.

## Comparison to Threads and Fibers

- **Threads**: preemptive, OS-scheduled, truly parallel on multiple cores, with shared mutable memory and all its hazards (races, locks). A generator has none of this — single-threaded, no parallelism, no preemption.
- **Fibers / green threads**: cooperative like generators, but typically the *runtime* schedules them and they can suspend from *anywhere* in the call stack, not only at syntactic `yield` points in the top frame. Generators are "stackless" coroutines: they can only suspend in their own body, not from a function they call (which is why `await` had to be a language feature, not just a generator). Fibers are "stackful."

The mental model: a generator is a **pausable function with a two-way channel**. Iterators are the simple use; async drivers and effect systems are the powerful one; `async`/`await` is the same idea promoted into the language with promises as the fixed transport.

## See Also
- [[async-await]]
- [[promises-internals]]
- [[03-computer-systems/concurrency-and-parallelism/csp-and-channels|CSP and Channels]]
- [[03-computer-systems/concurrency-and-parallelism/threads-and-processes|Threads and Processes]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
