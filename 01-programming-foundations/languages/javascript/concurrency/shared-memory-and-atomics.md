# Shared Memory and Atomics

`SharedArrayBuffer` (SAB) is the one place where JavaScript deliberately abandons its share-nothing model: a block of bytes whose backing store is visible to multiple agents at once. `Atomics` is the small instruction set that makes concurrent access to that memory well-defined. Together they let Workers (and WebAssembly threads) operate on the same physical memory without copying — at the cost of inheriting every hazard of real shared-memory concurrency.

## What is shared, and what is not

A normal `ArrayBuffer` transferred to a Worker is *moved* — only one agent owns it. A `SharedArrayBuffer` is *referenced* by every agent it is sent to; `postMessage` ships a handle, not a copy, and the bytes are not detached on the sender. You overlay a typed array (`Int32Array`, `BigInt64Array`, etc.) on it to read and write. Only the raw bytes are shared — JS objects, the heap, and GC are never shared. So shared memory is purely a numeric scratchpad; structured data must be hand-encoded into it.

```
agent A          agent B
   │   ┌──────────────────┐
   └──▶│  SharedArrayBuffer │◀── └   same bytes, two views
       │  Int32Array view   │
       └──────────────────┘
```

## The JS memory model

Once two threads read and write the same address, you have entered the realm of a formal **memory model**. ECMAScript adopted one (modeled on C++11). The key facts:

- **Non-atomic accesses** (ordinary `arr[i] = x`) participate in races. Two unsynchronized accesses to the same location where at least one is a write is a **data race**, and a racy read may observe a *garbage* value — not necessarily the old or new value, because the compiler and CPU may tear, reorder, or cache. JS does not crash on a data race (memory safety holds), but the *value* is unspecified.
- **Atomic accesses** via `Atomics.load`/`store`/`add`/`compareExchange`/etc. are **sequentially consistent** with respect to each other: there is a single total order of all atomic operations that every agent agrees on, and within a thread program order is preserved. SC is the strongest, easiest-to-reason-about ordering — no relaxed/acquire-release tiers are exposed to JS (WASM has more).
- Atomic operations also act as **memory fences**: they bound the reordering of surrounding non-atomic accesses, which is how you publish non-atomic data safely (write the payload non-atomically, then `Atomics.store` a "ready" flag; the reader spins on the flag atomically, and once it sees the flag the payload writes are guaranteed visible).

This is the same acquire/release publication discipline as in C/C++ and the same hardware reality of out-of-order execution and store buffers — JS just hides the weaker orderings.

## Atomics.wait / notify — blocking in JavaScript

`Atomics.wait(i32, index, expected, timeout)` is remarkable: it **blocks the calling thread** until another agent calls `Atomics.notify(i32, index, count)` on the same slot, or it times out. This is a futex — a building block for mutexes, semaphores, and condition variables, entirely in user space when uncontended.

The catch: `Atomics.wait` throws on the **main thread** (it would freeze the UI and the event loop). It is legal only inside Workers. For the main thread there is `Atomics.waitAsync`, which returns a promise instead of blocking — bridging the futex to the event loop. The idiomatic pattern: build a spinlock or a lock from `compareExchange`, but fall back to `wait`/`notify` under contention to avoid burning a core spinning.

```js
// acquire (worker): CAS the lock word from 0(free) to 1(held)
while (Atomics.compareExchange(lock, 0, 0, 1) !== 0) {
  Atomics.wait(lock, 0, 1);        // sleep until released
}
// ... critical section ...
Atomics.store(lock, 0, 0);
Atomics.notify(lock, 0, 1);        // wake one waiter
```

## Why it disappeared, and COOP/COEP

SAB shipped in 2017, then was **disabled in all browsers in January 2018** because **Spectre** showed that any high-resolution timer plus shared memory lets attackers build a timing side-channel to read cross-origin memory. SAB returned only behind **cross-origin isolation**: the document must send `Cross-Origin-Opener-Policy: same-origin` (COOP, severs the link to other browsing contexts) *and* `Cross-Origin-Embedder-Policy: require-corp` (COEP, forces every subresource to opt in). Only then does `crossOriginIsolated === true` and `SharedArrayBuffer` become constructible. This is the price of admission and a frequent deployment surprise: third-party embeds, ads, and non-CORP assets break under COEP, so enabling SAB is an architectural decision, not a flag flip.

## Use with WebAssembly threads

The premier consumer of SAB is **WebAssembly threading**. WASM linear memory backed by a SAB lets multiple Workers run the same module against shared memory, which is how Emscripten compiles pthreads — each pthread is a Worker, the C heap is the shared buffer, and C atomics map to JS/WASM atomics. This is what brings real multithreaded native code (game engines, ffmpeg, SQLite WAL) to the browser.

## Trade-offs and philosophy

SAB trades the actor model's safety for raw speed and zero-copy sharing. You regain data races, the need for explicit synchronization, and the obligation to reason about visibility and ordering — the hardest part of systems programming, now in JS. Use it only when message-passing copy costs are demonstrably the bottleneck, prefer single-producer/single-consumer ring buffers and lock-free structures over general locks, and treat any non-atomic access to shared slots as a latent bug. The mental model to keep: *shared memory is fast and dangerous; messages are slow and safe* — and most application code should stay on the safe side.

## See Also
- [[web-workers]] — the agents that share the buffer
- [[03-computer-systems/concurrency-and-parallelism/atomic-operations|Atomic Operations]]
- [[03-computer-systems/concurrency-and-parallelism/memory-ordering|Memory Ordering]]
- [[03-computer-systems/concurrency-and-parallelism/race-conditions|Race Conditions]]
- [[03-computer-systems/concurrency-and-parallelism/lock-free-programming|Lock-Free Programming]]
- [[03-computer-systems/computer-architecture/memory-models|Memory Models]]
