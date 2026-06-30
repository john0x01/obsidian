# Timers and Microtasks

Node interleaves four scheduling mechanisms with subtly different rules: `process.nextTick`, Promise microtasks, `setTimeout`/`setInterval`, and `setImmediate`. The libuv loop only knows about timers and immediates (they map to phases); the other two are **Node-injected queues drained between every callback**. Getting the ordering right is the difference between code that is merely correct and code whose latency and fairness you can actually reason about.

## Two Macrotask Sources, Two Micro-ish Queues

- **`setTimeout(fn, ms)` / `setInterval`** → libuv **timers** phase. `ms` is a *minimum* delay; the real fire time depends on loop load. Minimum effective delay is 1ms (0 is coerced up to 1).
- **`setImmediate(fn)`** → libuv **check** phase, i.e. right after `poll`.
- **`process.nextTick(fn)`** → the **nextTick queue**, a Node construct, *not* a libuv phase.
- **Promise reactions / `queueMicrotask` / `await` continuations** → V8's **microtask (Job) queue**.

The two non-libuv queues are not macrotasks. They are drained **after the current JS operation completes and before the loop is allowed to proceed to the next callback or next phase**.

## The Drain Order Between Callbacks

After any callback returns to Node (and after the initial top-level script), Node drains in this strict order:

```
current callback finishes
  -> drain ENTIRE nextTick queue   (FIFO; includes ticks queued while draining)
  -> drain ENTIRE microtask queue  (Promise jobs; includes jobs queued while draining)
  -> only now: next callback / next libuv phase
```

So **`process.nextTick` always beats Promise microtasks**, and *both* run before any timer or immediate. This drain happens at every boundary — between two timer callbacks, between two I/O callbacks, after the check phase, everywhere — not just once per loop iteration.

```js
setImmediate(() => console.log('immediate'));
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
// sync, nextTick, promise, then (timeout|immediate in unstable order at top level)
```

## The Classic Gotchas

**`setTimeout(0)` vs `setImmediate` at the top level is non-deterministic.** Both are queued before the loop starts; whether the first iteration's *timers* phase sees the 1ms timer as "due" depends on sub-millisecond startup jitter. You may get either order across runs.

**Inside an I/O callback, `setImmediate` always precedes `setTimeout(0)`.** You are already executing in the poll phase; the very next phase is *check*, so the immediate fires this iteration while the timer waits for the *next* iteration's timers phase. This is the only reliable ordering between the two, and the reason `setImmediate` is the correct "after current I/O, before more timers" primitive.

```js
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate')); // always first here
});
```

**`await` is a microtask, so it outruns timers.** `await null` does not yield to I/O — it schedules a microtask, which drains before the loop advances. A loop of `await`s over already-resolved values can stall timer and I/O processing just like a `.then` chain.

## Starving the Loop With nextTick

Because the nextTick queue is drained **to completion** before the loop may proceed, a callback that recursively re-queues `process.nextTick` creates a queue that never empties — the loop **never reaches poll**, so timers never fire, I/O never completes, the process appears hung at ~100% CPU.

```js
function starve() { process.nextTick(starve); } // loop never advances
```

A recursive `setImmediate` does the opposite: each one yields back to the loop, so I/O and timers interleave fairly. This is the canonical guidance — **prefer `setImmediate` for "do more work later but stay responsive"; reserve `nextTick` for "run this before anything else, but once."** Promise microtask recursion has the same starvation hazard as nextTick, just one priority level lower.

## Why nextTick Exists At All

Two legitimate uses: (1) let a function return *before* its callback fires so callers can attach handlers (e.g. emit an `'error'` on the next tick so a synchronously-attached listener exists), preserving the "callbacks are always async" contract; (2) clean up or roll back state after the current operation but before any I/O. Its sky-high priority is exactly what makes it dangerous — it is a foot-gun dressed as a convenience.

## Senior Pitfalls

- Assuming `setTimeout(0)` is "immediate" — it is a full macrotask, minimum 1ms, behind both micro queues.
- Assuming microtasks respect timer delays — they ignore the loop entirely until drained.
- Using `nextTick` for "later" (you meant `setImmediate`) and accidentally starving I/O.
- Trusting top-level timer-vs-immediate ordering observed once; it is a race, not a rule.

## See Also
- [[libuv-and-event-loop]] — the phases these queues drain between
- [[runtime-architecture]] — V8 owns microtasks, Node owns nextTick
- [[thread-pool]] — completions that schedule these callbacks
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]] — microtask vs macrotask model
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]] — Promise reactions as jobs
- [[03-computer-systems/concurrency-and-parallelism/async-and-await|Async and Await]] — await as a microtask continuation
- [[01-programming-foundations/languages/javascript/asynchrony/promises-internals|Promises Internals]] — the microtask source
