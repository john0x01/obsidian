# Web Workers

JavaScript on the main thread is single-threaded: one call stack, one event loop, one heap servicing the UI. Web Workers are the platform's escape hatch from that constraint — OS-backed background threads that run a separate JS realm in true parallel, communicating only by message passing. They exist to keep the main thread responsive: CPU-bound work (parsing, crypto, compression, image processing) that would otherwise block input handling and rendering is moved off the thread that paints.

## The single-threaded model and what a Worker actually is

The "single-threaded" claim is about *script execution per realm*, not the engine as a whole. V8 already runs GC, compilation, and background JIT on helper threads; the browser runs compositing, networking, and many timers on separate threads. What is serialized is *your* JS plus DOM mutation, behind one event loop. A Worker spins up a fresh agent: its own event loop, its own global (`self`, a `WorkerGlobalScope`, not `window`), its own microtask queue, and — critically — its own heap. There is no shared object graph. A Worker cannot touch the DOM, `window`, `document`, or `localStorage`. It gets `fetch`, timers, `WebSocket`, `IndexedDB`, `OffscreenCanvas`, and importantly other Workers.

```
main thread (agent)        worker (agent)
┌───────────────┐  postMessage   ┌───────────────┐
│ heap A, loop A │ ────────────▶ │ heap B, loop B │
│ DOM, window    │ ◀──────────── │ self, no DOM   │
└───────────────┘   onmessage    └───────────────┘
```

## Message passing, structured clone, and the actor model

The only channel between agents is `postMessage` / `onmessage`. The payload is not passed by reference — by default it is *copied* using the **structured clone algorithm**, which is more capable than `JSON.stringify`: it handles cyclic references, `Map`, `Set`, `Date`, `RegExp`, `ArrayBuffer`, typed arrays, `Blob`, and `File`. It cannot clone functions, DOM nodes, or prototype chains (a class instance arrives as a plain object). Clone cost is O(size of graph) and synchronous on the sending side, so shipping a 50 MB object is a real main-thread stall — the copy itself blocks.

This "no shared mutable state, communicate by copying messages" design is the **actor model** in all but name: each Worker is an actor with private state and a mailbox (its event queue), reacting to asynchronous messages. The payoff is the absence of an entire class of bugs — no data races on ordinary JS values, no need for locks around the object graph, because nothing is shared. Determinism per actor; concurrency between actors. (Shared memory via `SharedArrayBuffer` deliberately breaks this model and reintroduces races — see the companion note.)

## Transferable objects

Copying is wasteful when you mean to hand off ownership. **Transferable objects** (`ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, `ImageBitmap`, `ReadableStream`) move in O(1) by transferring the underlying memory rather than copying it. You opt in via the transfer list:

```js
worker.postMessage(buf, [buf]); // buf is now neutered (byteLength 0) on the sender
```

After transfer the buffer is *detached* on the sender — accessing it throws or yields zero length. This is move semantics: exactly one agent owns the bytes at a time, preserving the no-races guarantee while eliminating the copy.

## Three flavors

- **Dedicated Worker** — owned by a single document/script (`new Worker(url)`). One-to-one channel. The workhorse.
- **Shared Worker** — one instance shared across tabs/windows of the same origin, addressed through `port`s (`new SharedWorker(url)`). Useful for a single WebSocket or coordination point across tabs; underused because of debugging friction and lifecycle subtlety.
- **Service Worker** — not a parallelism tool at all but a *programmable network proxy* that sits between page and network, enabling offline caching, push, and background sync. Event-driven, can be killed and resurrected by the browser at will, so it must hold no durable in-memory state. Conflating it with computation Workers is a common category error.

Module Workers (`new Worker(url, { type: 'module' })`) allow `import`; classic Workers use `importScripts`.

## Trade-offs, pitfalls, and philosophy

Workers are not free. Spawn cost is non-trivial (fresh realm, separate context) — for short tasks the startup plus serialization can dwarf the work, so pool and reuse Workers. The hard ceiling is the *serialization boundary*: chatty designs that round-trip large objects per message will spend more time cloning than computing. Design for coarse-grained, infrequent messages or transfer/shared memory. Debugging spans multiple realms; error propagation requires the `onerror`/`messageerror` handlers. And Workers buy *responsiveness*, not necessarily *throughput* — moving work off-thread frees the UI even if the total CPU is unchanged; true throughput gains need genuine parallel cores and apply only to parallelizable, CPU-bound work (Amdahl's law still rules).

The deeper philosophy: the web chose **share-nothing message passing** over shared-memory threading as the *default* precisely because it makes concurrency tractable for application authors. You reach for `SharedArrayBuffer` only when the message-passing tax is provably too high.

## See Also
- [[shared-memory-and-atomics]] — opting out of share-nothing
- [[runtime]] — the single event loop a Worker duplicates
- [[event-loops]] — each Worker has its own loop and queues
- [[03-computer-systems/concurrency-and-parallelism/actors|Actors]]
- [[03-computer-systems/concurrency-and-parallelism/threads-and-processes|Threads and Processes]]
- [[03-computer-systems/concurrency-and-parallelism/race-conditions|Race Conditions]]
- [[10-frameworks-and-stacks/node-js/concurrency/worker-threads|Node Worker Threads]] — the Node equivalent
